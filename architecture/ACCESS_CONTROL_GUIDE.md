# Access Control Guide

## Testing

### Unit Testing

- **AccessMiddleware**: Test access rule patterns
- **Controllers**: Test business logic (assume access verified)
- **No access testing in controller tests**

### Integration Testing

- **Full request flow**: Test middleware → controller → response
- **Authorization scenarios**: Test 403 responses for denied access
- **Role-based access**: Test different user roles

### Test Structure

```php
// Test AccessMiddleware
public function testOwnerOrAdminAccess() {
    $request = [
        'identity' => $user,
        'uri' => '/collections/123/edit'
    ];
    
    $middleware = new AccessMiddleware($repositories);
    $result = $middleware->handle($request);
    
    assert($result['authorized'] === true);
}

// Test Controller (no access checking)
public function testEditBusinessLogic() {
    $request = [
        'identity' => $owner,
        'authorized_resource' => $collection,
        'validated' => ['name' => 'New Name']
    ];
    
    $controller = new CollectionController($services);
    $response = $controller->edit($request);
    
    assert($response['success'] === true);
}
```

## Troubleshooting

### Common Issues

1. **403 Forbidden Errors**
   - Check access configuration in `routes.json`
   - Verify user identity is set by `IdentityMiddleware`
   - Ensure route has correct `access.type` defined

2. **Resources Not Loading**
   - Verify repository is injected in `AccessMiddleware`
   - Check resource type mapping in `ConfigManager` (`resources:*` keys)
   - Ensure resource ID extraction is correct

3. **Missing targetUserId**
   - Check route URI pattern includes user ID parameter
   - Verify URI structure matches expected patterns
   - Ensure middleware execution order is correct

### Debug Steps

1. Check middleware execution order in `App.php`
2. Verify `$request['identity']` is set
3. Check `$request['access_uri']` for URI being processed
4. Examine middleware logs for authorization decisions
5. Test access patterns using verification scripts

## Security Considerations

### Fail-Closed Design
- Default to deny access
- Explicit allow patterns only
- No bypass mechanisms

### Admin Override
- Use sparingly and intentionally
- Document which resources allow admin override
- Consider owner-only for sensitive operations

### Resource Loading
- Middleware loads resources to prevent duplicate queries
- Controllers use pre-loaded data when available
- Fallback loading only when necessary

### Error Information
- 403: Authorization failure (don't reveal resource details)
- 404: Resource not found (safe to disclose)
- No information leakage about existence of resources

## Migration Checklist

When migrating a controller to the middleware-based access control pattern:

- [ ] Remove `AccessHelper::isOwnerOrAdmin()` calls from controller
- [ ] Remove manual admin checks (`$isAdmin = ...`)
- [ ] Remove manual authorization checks
- [ ] Add route to `routes.json` with appropriate `access` configuration
- [ ] Use `$request['authorized_resource']` in controller
- [ ] Use `$request['targetUserId']` for user-scoped operations
- [ ] Update tests to not test authorization (only business logic)
- [ ] Verify 403 responses come from middleware, not controller
- [ ] Document any special authorization requirements

## Best Practices

### Do
- Trust middleware has verified authorization
- Use pre-loaded resources from `$request['authorized_resource']`
- Return 404 for "not found", let middleware handle 403
- Keep authorization logic in middleware only
- Use ConfigManager for configuration values (roles, fields, resources)

### Don't
- Check authorization in controllers
- Duplicate access logic in multiple places
- Use hardcoded strings for roles/resources (use ConfigManager)
- Bypass middleware authorization checks
- Return 403 from controllers (use middleware)

## Quick Reference

### Access Rule Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| Public | No auth required | `/login` |
| Authenticated | Must be logged in | `/dashboard` |
| Admin-only | Must be admin | `/admin/users` |
| Owner-or-admin | Owner or admin | `/collections/123/edit` |
| Owner-only | Owner only, no admin | `/user/123/edit-password` |

### Request Context Keys

| Key | Description | Set By |
|-----|-------------|--------|
| `identity` | User identity object | IdentityMiddleware |
| `authorized_resource` | Pre-loaded resource | AccessMiddleware |
| `targetUserId` | Target user for operation | AccessMiddleware |
| `access` | Access metadata | AccessMiddleware |

---

**See Also:**
- [ACCESS_CONTROL_OVERVIEW.md](ACCESS_CONTROL_OVERVIEW.md) - High-level concepts and principles
- [ACCESS_CONTROL_IMPLEMENTATION.md](ACCESS_CONTROL_IMPLEMENTATION.md) - Implementation details and patterns

**Document Version:** 1.1
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete