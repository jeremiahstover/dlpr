# Route Alias System

This document explains the route alias system in the Application, which provides URI shortcuts and legacy URL support while maintaining clean routing logic.

## Overview

The route alias system allows mapping one URI to another, enabling:
- Shorter, more memorable URLs
- Legacy URL support for backward compatibility
- Cleaner URI structures without changing application logic

## Implementation

Aliases are defined in `Data/aliases.json` and processed by the `Router` class in `App/Routes/Router.php`.

### Alias Definition Format

The `aliases.json` file contains a simple key-value mapping:

```json
{
    "/": "/page/home",
    "/about": "/page/about",
    "/features": "/page/features",
    "/login": "/user/login",
    "/profile": "/user",
    "/profile[/{id}]": "/user[/{id}]"
}
```

### Key Features

1. **Simple Mapping**: Direct URI-to-URI mapping
2. **Pattern Support**: Parameterized routes with optional parameters
3. **Legacy Support**: Maintains compatibility with old URLs
4. **Redirect-Free Resolution**: Aliases are resolved internally without HTTP redirects

## Alias Resolution Logic

The alias resolution occurs in the `Router::match()` method:

```php
// First, check if this URI is an alias and resolve it
if (isset($this->aliases[$uri])) {
    $alias = $this->aliases[$uri];

    // Support legacy-style controller mappings inside aliases.json
    if (is_array($alias) && isset($alias['controller'], $alias['method'])) {
        // Set API flag in request if provided
        if ($request !== null && $isApi) {
            $request['is_api'] = true;
        }

        return $alias;
    }

    if (is_string($alias)) {
        $uri = $alias;
    }
}
```

### Resolution Process

1. **Alias Lookup**: Check if the requested URI exists as a key in the aliases array
2. **Value Type Check**: Determine if the alias value is a direct mapping or legacy controller mapping
3. **Controller Mapping**: For legacy mappings, return controller configuration directly
4. **String Resolution**: For string mappings, replace the URI with the mapped value
5. **Continue Processing**: Proceed with normal route matching using the resolved URI

## Pattern Support

The alias system supports parameterized routes with optional parameters using bracket notation:

```json
"/profile[/{id}]": "/user[/{id}]",
"/profile[/{id}]/edit": "/user[/{id}]/edit"
```

### How Patterns Work

1. **Optional Parameters**: Brackets `[]` denote optional parameter segments
2. **Parameter Matching**: Parameters within braces `{}` are captured during matching
3. **Pattern Translation**: Routes with patterns are converted to regular expressions for matching

### Pattern Resolution Example

When a request comes in for `/profile/123`:
1. The alias system doesn't directly match `/profile/123` (it only has `/profile[/{id}]`)
2. The request proceeds to normal route matching
3. The pattern matching system converts `/profile[/{id}]` to a regex pattern
4. The regex matches `/profile/123` and captures the `id` parameter
5. The route resolves to `/user[/{id}]` which becomes `/user/123` during resolution

## Precedence Rules

The alias resolution system follows specific precedence rules:

### Alias vs. Direct Routes

1. **Direct Route Check First**: The system first checks for exact matches in the main routes
2. **Alias Resolution Second**: If no direct route is found, aliases are checked
3. **Pattern Matching Last**: If neither direct routes nor aliases match, pattern matching is attempted

### Conflict Resolution

When an alias conflicts with a direct route:
- Direct routes take precedence over aliases
- This prevents accidental overriding of explicit route definitions

## Legacy Controller Mappings

The alias system supports legacy controller mappings for backward compatibility:

```json
{
    "/old-endpoint": {
        "controller": "OldController",
        "method": "oldMethod"
    }
}
```

### Processing Legacy Mappings

When an alias value is an array with `controller` and `method` keys:
1. The system immediately returns this configuration without further processing
2. This bypasses normal route matching
3. API flag is set if applicable

## API Handling

The alias system integrates with API route handling:

### API Prefix Stripping

API routes with the `/api/` prefix are handled specially:

```php
// Handle /api/* prefix (strip /api and look up the rest)
if (strpos($uri, '/api/') === 0) {
    $isApi = true;
    $uri = str_replace('/api/', '/', $uri);
}
```

### Alias Resolution with API Routes

When resolving aliases for API routes:
1. The `/api/` prefix is stripped before alias lookup
2. The alias is resolved using the stripped URI
3. The API flag is preserved in the request data

## Security Considerations

### Path Traversal Prevention

The alias system inherently prevents path traversal because:
- All aliases are predefined in `aliases.json`
- No user input directly influences alias resolution
- Aliases are resolved to predetermined values

### Injection Prevention

- Alias values are not interpreted as code
- All URI manipulation is controlled and validated
- No dynamic alias creation from user input

## Performance Implications

### Memory Usage

- Aliases are loaded once during router initialization
- JSON parsing happens only once per application lifecycle
- Minimal memory overhead for alias storage

### Lookup Performance

- Alias lookup is an O(1) hash table operation
- No database queries required for alias resolution
- Fast string replacement for resolved aliases

### Pattern Matching Overhead

- Pattern-based aliases require regex compilation
- Compiled patterns are cached for reuse
- Minimal performance impact for typical usage

## Maintenance Considerations

### Adding New Aliases

To add a new alias:
1. Add the mapping to `Data/aliases.json`
2. No application restart required (file is read on each request)
3. Test the new alias to ensure proper resolution

### Removing Old Aliases

To remove deprecated aliases:
1. Remove the mapping from `Data/aliases.json`
2. Verify no internal links reference the alias
3. Monitor logs for 404 errors after removal

## Debugging and Troubleshooting

### Common Issues

1. **Alias Not Resolving**: Check `aliases.json` syntax and exact URI matching
2. **Circular References**: Ensure aliases don't create infinite loops
3. **Pattern Mismatch**: Verify pattern syntax matches expected format

### Diagnostic Techniques

1. **Log Inspection**: Check router logs for alias resolution information
2. **Direct Testing**: Test aliases through direct browser requests
3. **Unit Tests**: Use existing router tests to validate alias behavior

## Integration with Other Systems

### Middleware Compatibility

The alias system works seamlessly with the middleware pipeline:
- Aliases are resolved before middleware execution
- Request data includes resolved URIs
- API flags are properly propagated

### Template Integration

- Templates can reference aliased URLs directly
- URL generation functions can utilize alias mappings
- No special handling required in presentation layer

## Best Practices

### Alias Design

1. **Consistent Naming**: Use consistent naming conventions for aliases
2. **Logical Grouping**: Group related aliases together in the JSON file
3. **Documentation**: Comment complex or non-obvious aliases
4. **Versioning**: Consider versioning strategy for major alias changes

### Performance Optimization

1. **Limit Pattern Usage**: Use patterns sparingly as they require regex processing
2. **Cache Warm-Up**: Consider preloading frequently used aliases
3. **Monitor Usage**: Track which aliases are actually used to identify candidates for removal

### Backward Compatibility

1. **Maintain Legacy Routes**: Keep old aliases for extended periods
2. **Gradual Migration**: Migrate internal links to use canonical URLs
3. **Announce Changes**: Communicate alias deprecations to stakeholders
---

**Related Docs:**
- [ROUTE_MAPPING.md](ROUTE_MAPPING.md) - Route matching and configuration
- [ENTRY_POINTS.md](ENTRY_POINTS.md) - Request routing overview
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - URI type classification

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
