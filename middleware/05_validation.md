# Tier 05: InputValidation

**Purpose**: Validate and sanitize input data against predefined schemas, casting values to appropriate types and rejecting malformed requests before they reach controllers.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['validated']` | `array` | Sanitized and type-cast input data matching the route's schema |

## Responsibilities

1. **Schema-Based Validation**: Applies route-specific validation schemas
2. **Type Casting**: Converts input values to appropriate PHP types (int, bool, string)
3. **Format Validation**: Validates email addresses, URLs, and patterns
4. **Range Checking**: Enforces min/max constraints on strings and numbers
5. **Choice Validation**: Ensures values are within allowed sets
6. **Multi-Source Merging**: Combines data from query params, POST data, and URI params

## Validation Schema Structure

Validation schemas are keyed by `METHOD:URI` pattern:

```php
$this->schemas = [
    'POST:/studies/save' => [
        'title' => ['required' => true, 'type' => 'string', 'min' => 1, 'max' => 255],
        'type' => ['required' => true, 'type' => 'string', 'choices' => ['verse', 'topic', 'reference', 'collection']],
        'reference' => ['required' => false, 'type' => 'string', 'max' => 500],
        'id' => ['required' => false, 'type' => 'numeric', 'min' => 1],
    ],
];
```

### Schema Field Rules

| Rule | Type | Description |
|------|------|-------------|
| `required` | `bool` | Field must be present and non-empty |
| `type` | `string` | Data type: `string`, `numeric`, `bool`, `array` |
| `format` | `string` | Special format: `email`, `url` |
| `min` | `int` | Minimum length (strings) or value (numbers) |
| `max` | `int` | Maximum length (strings) or value (numbers) |
| `choices` | `array` | Allowed values (enum-like validation) |
| `pattern` | `string` | Regex pattern for validation |

## Type Casting Rules

### Numeric Type

```php
case 'numeric':
    if (!is_numeric($value)) {
        return ['error' => "$field must be numeric", 'value' => null];
    }
    return ['error' => null, 'value' => is_float($value) ? (float)$value : (int)$value];
```

Input `42` → Output `42` (int)
Input `3.14` → Output `3.14` (float)
Input `"123"` → Output `123` (int)

### String Type

```php
case 'string':
    $trimmed = trim((string)$value);
    if ($field === 'email') {
        $trimmed = strtolower($trimmed);  // Normalize emails
    }
    return ['error' => null, 'value' => $trimmed];
```

Input `"  Hello  "` → Output `"Hello"`
Input `"User@Example.COM"` → Output `"user@example.com"` (email field)

### Boolean Type

```php
case 'bool':
    if (is_bool($value)) return $value;
    if (is_numeric($value)) return (bool)$value;
    if (is_string($value)) {
        $lower = strtolower($value);
        if (in_array($lower, ['true', '1', 'yes', 'on'])) return true;
        if (in_array($lower, ['false', '0', 'no', 'off', ''])) return false;
    }
    return (bool)$value;
```

| Input | Output |
|-------|--------|
| `true` | `true` |
| `1` | `true` |
| `"yes"` | `true` |
| `"on"` | `true` |
| `false` | `false` |
| `0` | `false` |
| `"no"` | `false` |
| `""` | `false` |

## Data Source Priority

Input data is merged from multiple sources, with later sources taking precedence:

```php
private function extractInputData(array $request): array {
    $data = [];
    
    // 1. Query params (lowest priority)
    if (isset($request['query'])) {
        $data = array_merge($data, $request['query']);
    }
    
    // 2. URI params
    if (isset($request['uri_params'])) {
        $data = array_merge($data, $request['uri_params']);
    }
    
    // 3. POST data (highest priority)
    if (isset($request['post'])) {
        $data = array_merge($data, $request['post']);
    }
    
    return $data;
}
```

## Schema Matching

### Exact Match

```php
$key = "POST:/studies/save";
if (isset($this->schemas[$key])) {
    return $this->schemas[$key];
}
```

### Pattern Match

For routes with parameters like `/studies/{id}/save`:

```php
foreach ($this->schemas as $pattern => $schema) {
    list($schemaMethod, $schemaUri) = explode(':', $pattern, 2);
    
    // Convert route pattern to regex
    $regex = preg_replace('/\{[^}]+\}/', '[^/]+', $schemaUri);
    $regex = '#^' . $regex . '$#';
    
    if (preg_match($regex, $uri)) {
        return $schema;
    }
}
```

## Study Reference Validation

Special validation ensures studies have at least one reference field:

```php
private function validateStudyReferenceRequirement(array $request, array $validated): void {
    if (preg_match('#^/studies(?:/\d+)?/save$#', $uri) !== 1) {
        return;
    }

    $reference = trim((string)($validated['reference'] ?? ''));
    $referenceVerse = trim((string)($validated['reference_verse'] ?? ''));
    $referenceTopic = trim((string)($validated['reference_topic'] ?? ''));
    $referenceCollection = trim((string)($validated['reference_collection'] ?? ''));

    if ($reference === '' && $referenceVerse === '' && $referenceTopic === '' && $referenceCollection === '') {
        throw new \RuntimeException('Validation failed: reference is required', 400);
    }
}
```

## Focused Times Normalization

The middleware normalizes the `focused_times` array structure:

```php
// Input from form
[
    ['name' => 'morning', 'time' => '5:00am'],
    ['name' => 'evening', 'time' => '7:00pm'],
]

// Output after normalization
[
    'morning' => '5:00am',
    'evening' => '7:00pm',
]
```

## Error Handling

| Exception | Code | Condition |
|-----------|------|-----------|
| `RuntimeException` | 400 | Required field missing or empty |
| `RuntimeException` | 400 | Type mismatch (non-numeric value for numeric field) |
| `RuntimeException` | 400 | Format validation failure (invalid email) |
| `RuntimeException` | 400 | Range constraint violation (string too long) |
| `RuntimeException` | 400 | Choice validation failure (value not in allowed set) |
| `RuntimeException` | 400 | Pattern validation failure |

## Error Message Format

Validation errors are aggregated and reported as a comma-separated list:

```php
throw new \RuntimeException(
    'Validation failed: ' . implode(', ', $validationResult['errors']),
    400
);
```

Example: `"Validation failed: title is required, email must be a valid email address"`

## Fallback Validation

For routes without defined schemas, basic ID validation is applied:

```php
private function validateBasicIds(array $request): array {
    // Fields ending with _id or named id are validated as numeric
    if (str_ends_with($key, '_id') || $key === 'id') {
        if (!is_numeric($value)) {
            throw new \RuntimeException('Validation failed: ' . $key . ' must be numeric', 400);
        }
        $validated[$key] = (int)$value;
    }
}
```

## Downstream Usage

Controllers access validated data via `$request['validated']`:

```php
public function save($app, array $request): array {
    // Data is already validated and type-cast
    $title = $request['validated']['title'];        // string
    $studyId = $request['validated']['id'] ?? null;  // int|null
    $type = $request['validated']['type'];          // string
    
    // No need for manual validation or type casting
}
```

## Schema Configuration

Schemas are initialized in the constructor with config-aware values:

```php
private function initializeSchemas(array $config): void {
    $studyTypes = $config['study_types'] ?? ['verse', 'topic', 'reference', 'collection'];
    
    $interfaceMap = $config['interface_map'] ?? [
        '0' => 'guest', '1' => 'focused', '2' => 'normal', 
        '3' => 'power', '4' => 'programmer', '9' => 'admin',
    ];
}
```

## Dependencies

- Config values from `$request['config']` for `study_types` and `interface_map`

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 04: HttpMethodOverride](04_http_override.md)
- [Tier 06: UserContext](06_context.md)
