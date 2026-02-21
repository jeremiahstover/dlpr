# Tier 04: HttpMethodOverride

**Purpose**: Normalize HTTP verbs for HTML form submissions by supporting the `_method` field override pattern.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['method']` | `string` | Modified HTTP method when override is applied |
| `$request['original_method']` | `string` | Preserved original method (`POST`) for logging/debugging |

## The Problem

HTML forms only support `GET` and `POST` methods. RESTful APIs and modern web applications require additional verbs:

- `PUT` - Update resources
- `PATCH` - Partial resource updates  
- `DELETE` - Remove resources

## The Solution

The `_method` field pattern allows forms to specify their intended HTTP verb:

```html
<form action="/studies/42" method="POST">
    <input type="hidden" name="_method" value="PUT">
    <!-- form fields -->
</form>
```

This middleware detects the `_method` field and updates `$request['method']` accordingly.

## Allowed Overrides

Only these HTTP methods can be specified via `_method`:

```php
private const ALLOWED_OVERRIDES = ['PUT', 'PATCH', 'DELETE'];
```

## Processing Logic

```php
public function handle(array $request): array {
    $method = strtoupper((string)($request['method'] ?? 'GET'));

    // Only process POST requests
    if ($method !== 'POST') {
        return $request;
    }

    // Extract _method from POST data
    $override = $request['post']['_method'] ?? null;
    if (!is_string($override) || $override === '') {
        return $request;
    }

    // Normalize and validate
    $override = strtoupper(trim($override));
    if (!in_array($override, self::ALLOWED_OVERRIDES, true)) {
        return $request;
    }

    // Apply override
    $request['method'] = $override;
    $request['original_method'] = 'POST';

    return $request;
}
```

## Flow Diagram

```
Request arrives
       │
       ▼
┌──────────────────────┐
│ Is method POST?      │─No──► Return unchanged
└──────────────────────┘
       │ Yes
       ▼
┌──────────────────────┐
│ Is _method field     │─No──► Return unchanged
│ present in POST?     │
└──────────────────────┘
       │ Yes
       ▼
┌──────────────────────┐
│ Is _method value in  │─No──► Return unchanged
│ ALLOWED_OVERRIDES?   │
└──────────────────────┘
       │ Yes
       ▼
┌──────────────────────┐
│ Set $request['method']│
│ to override value    │
└──────────────────────┘
       │
       ▼
┌──────────────────────┐
│ Set $request['       │
│ original_method']     │
│ to 'POST'            │
└──────────────────────┘
       │
       ▼
   Continue
```

## Usage Examples

### DELETE Request via Form

```html
<form action="/studies/42" method="POST">
    <input type="hidden" name="_method" value="DELETE">
    <input type="hidden" name="csrf_token" value="...">
    <button type="submit">Delete Study</button>
</form>
```

Result: `$request['method']` = `'DELETE'`

### PUT Request via Form

```html
<form action="/studies/42" method="POST">
    <input type="hidden" name="_method" value="PUT">
    <input type="text" name="title" value="Genesis 1">
    <button type="submit">Update Study</button>
</form>
```

Result: `$request['method']` = `'PUT'`

### JavaScript/AJAX Usage

```javascript
fetch('/studies/42', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: '_method=PUT&title=Genesis%201'
});
```

## Downstream Impact

The method override affects:

| Component | Impact |
|-----------|--------|
| Router | Matches routes based on overridden method |
| InputValidation | Selects validation schema using overridden method |
| CsrfProtection | Applies CSRF validation to all state-changing methods |

## Router Integration

The router matches routes using the potentially modified method:

```php
// routes.json
{
    "/studies/{id}": {
        "controller": "StudiesController",
        "method": "update",
        // This matches PUT /studies/42
    }
}

// Router matches based on $request['method'] after middleware processing
$route = $this->router->match($request['uri'], $request);
```

## Security Considerations

- **Only POST requests can be overridden**: Prevents malicious links from triggering state changes
- **Explicit allowed list**: Only `PUT`, `PATCH`, `DELETE` are permitted
- **Case-insensitive matching**: `put`, `Put`, `PUT` are all valid
- **Whitespace tolerance**: Trims the override value before validation

## Error Handling

HttpMethodOverride does not throw exceptions. Invalid override values are silently ignored and the original `POST` method is preserved.

## Dependencies

None. This middleware has no external dependencies.

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 03: AccessMiddleware](03_access.md)
- [Tier 05: InputValidation](05_validation.md)
