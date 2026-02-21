# Access Control Overview

## Overview

Access control is enforced entirely in middleware before controller execution. Controllers trust that authorization has been verified by the middleware layer and focus solely on business logic.

## Architecture Principles

### 1. Separation of Concerns
- **Middleware Layer**: Handles all authorization and access control
- **Controller Layer**: Executes business logic only, trusting middleware has verified access
- **No Cross-Layer Dependencies**: Controllers never import or call middleware classes

### 2. Fail-Closed Security
- All routes default to denied access
- Explicit permission required for every route
- Clear distinction between authentication and authorization

### 3. Centralized Authorization
- Single source of truth for access rules in middleware
- Consistent patterns across all resources
- Easy to audit and modify access policies

## Authorization Flow

```
Request → IdentityMiddleware → AccessMiddleware → Controller → Response
    ↓              ↓                ↓              ↓
  Extract      Set User       Check Access    Execute
  Identity     Identity       Rules           Logic
```

### Detailed Flow

1. **Request Arrives**
   - Request enters middleware stack

2. **IdentityMiddleware**
   - Extracts user identity from session/token
   - Sets `$request['identity']` with user data
   - Handles authentication (login/logout)

3. **AccessMiddleware**
   - Analyzes request URI and method
   - Retrieves access configuration from `routes.json` via `RoutesRepository`
   - Enforces authorization policies based on route `access.type`
   - Sets context in `$request`:
     - `$request['authorized_resource']`: Loaded resource (if applicable)
     - `$request['targetUserId']`: Target user ID for user-scoped operations
     - `$request['access']`: Access metadata (role, permissions)
   - Throws `RuntimeException('Forbidden', 403)` if access denied

4. **Controller Execution**
   - Trust middleware has verified authorization
   - Use pre-loaded resources from `$request['authorized_resource']`
   - Use target user ID from `$request['targetUserId']`
   - Focus on business logic only

5. **Response**
   - Controller returns response data
   - ResponseBuilder formats output
   - ErrorHandler catches exceptions

## Access Rules

Access rules are defined in `routes.json` and enforced by `AccessMiddleware`. The middleware reads route configuration via `RoutesRepository::getAccessForUri()`.

### Rule Types

#### 1. Public Routes
- No authentication required
- Accessible by anyone
- Examples: Login page, public API endpoints

#### 2. Authenticated-Only Routes
- User must be logged in
- Any role (user, admin) can access
- Examples: User dashboard, profile viewing

#### 3. Admin-Only Routes
- User must have admin role
- Also allows interface level 9 (system interface) bypass
- Examples: User management, system settings

#### 4. Owner-or-Admin Routes
- User must be resource owner OR admin
- Admin can access any resource
- Interface level 9 also bypasses ownership check
- Examples: Edit own content, manage own schedules

#### 5. Owner-Only Routes (Strict)
- Only the resource owner can access
- Admin cannot override
- Examples: Password changes, sensitive personal data

## Constants Configuration

All configuration values are managed through `ConfigManager` using dot-notation keys:

```php
// Roles
$config->get('roles:admin', 'admin');      // Admin role identifier
$config->get('roles:user', 'user');        // User role identifier
$config->get('roles:guest', 'guest');      // Guest role identifier

// Fields
$config->get('fields:user_id', 'user_id'); // User ID field name
$config->get('fields:owner', 'owner');     // Owner field name

// Resources
$config->get('resources:studies', 'studies');         // Studies resource
$config->get('resources:collections', 'collections'); // Collections resource
$config->get('resources:schedules', 'schedules');     // Schedules resource
$config->get('resources:notifications', 'notifications'); // Notifications resource

// Templates
$config->get('templates:collections_list', 'Collections/list');
$config->get('templates:user_view', 'User/view');
```

## Access Configuration Source

Access rules are defined in `routes.json` and read via `RoutesRepository::getAccessForUri()`. Example route configuration:

```json
{
  "/collections/{id}/edit": {
    "controller": "CollectionController",
    "method": "edit",
    "access": {
      "type": "owner_or_admin",
      "resource": "collections",
      "owner_field": "owner"
    }
  }
}
```

### Access Types

- `public`: No authentication required
- `authenticated_only`: Must be logged in
- `admin_only`: Must have admin role
- `owner_or_admin`: Resource owner or admin can access
- `owner_only`: Only resource owner (no admin override)

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

## Future Enhancements

### Planned Improvements
1. **Permission System**: Granular permissions beyond roles
2. **Resource Groups**: Batch authorization for related resources
3. **Dynamic Rules**: Database-driven access policies
4. **Audit Logging**: Track access decisions for compliance

### Extension Points
1. Custom access rule types
2. Plugin-based authorization
3. External identity providers
4. Rate limiting integration

---

**See Also:**
- [ACCESS_CONTROL_IMPLEMENTATION.md](ACCESS_CONTROL_IMPLEMENTATION.md) - Implementation details and patterns
- [ACCESS_CONTROL_GUIDE.md](ACCESS_CONTROL_GUIDE.md) - Testing, migration, and troubleshooting

**Document Version:** 1.0  
**Last Updated:** 2024-01-07  
**Status:** Accurate and Complete
