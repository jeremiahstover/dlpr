# Authorization Current State Analysis

## Overview

Authorization in Application is now centralized in middleware. All access control is enforced before controllers execute, following the middleware-based access control pattern. Controllers trust that authorization has been verified and focus solely on business logic.

## Current Authorization Implementation

### 1. AccessMiddleware Class

**Location:** `App/Logic/Middleware/AccessMiddleware.php`

**Responsibilities:**
- Reads access configuration from `routes.json` via `RoutesRepository`
- Enforces route access requirements (admin-only, authenticated, owner-or-admin, etc.)
- Loads and validates resource ownership for owner-based access
- Sets `$request['authorized']`, `$request['authorized_resource']`, `$request['targetUserId']`
- Throws `RuntimeException('Forbidden', 403)` for unauthorized access

**Access Types Supported:**
- `public` - No authentication required
- `authenticated_only` - Must be logged in
- `admin_only` - Must have admin role
- `owner_or_admin` - Resource owner or admin can access
- `owner_only` - Only resource owner (no admin override)

**Configuration Source:**
Access rules are defined in `routes.json`:
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

### 2. Controller Patterns

Controllers no longer perform authorization checks. They trust middleware has verified access.

#### Resource Controller Example
```php
public function edit(array $request): array {
    // Access already verified by middleware
    $collection = $request['authorized_resource'];
    
    if (!$collection) {
        return ResponseBuilder::notFound('Collection not found');
    }
    
    // Business logic only
    return ResponseBuilder::success([
        'collection' => $collection
    ]);
}
```

#### User-Scoped Controller Example
```php
public function view(array $request): array {
    // targetUserId set by middleware (handles ownership)
    $targetUserId = $request['targetUserId'] ?? $request['identity']->id;
    
    $user = $this->userService->getUserById($targetUserId);
    // ... business logic
}
```

### 3. AccessHelper Class (Legacy)

**Location:** `App/Logic/Helpers/AccessHelper.php`

**Status:** Legacy utility, no longer used in controllers.

**Note:** The `isOwnerOrAdmin()` method was previously used in controllers but has been removed as part of the middleware-based access control migration. All authorization logic now lives in `AccessMiddleware`.

## Authorization Flow

```
Request → IdentityMiddleware → AccessMiddleware → Controller
              ↓                        ↓                ↓
        Set identity            Check access      Execute logic
        from session            via routes.json   (trust middleware)
```

## Benefits of Current Approach

1. **Separation of Concerns**: Controllers focus on business logic only
2. **Consistency**: All access rules in one place (routes.json)
3. **Maintainability**: Easy to audit and modify access policies
4. **Testability**: Separate testing of authorization and business logic
5. **Security**: Centralized, fail-closed access control
6. **Performance**: Resource loading cached in middleware

## Migration Complete

The migration from controller-based to middleware-based access control is complete. All controllers have been updated to:
- Remove `AccessHelper::isOwnerOrAdmin()` calls
- Remove manual admin checks
- Use `$request['authorized_resource']` for pre-loaded resources
- Use `$request['targetUserId']` for user-scoped operations
- Trust middleware for all authorization

---

**Document Version:** 2.0  
**Last Updated:** 2025-02-08  
**Status:** Accurate and Complete

**See Also:** [ACCESS_CONTROL_OVERVIEW.md](ACCESS_CONTROL_OVERVIEW.md) - Current middleware-based access control architecture