# Authorization Architecture (Implemented)

## Overview

This document describes the middleware-based authorization architecture that has been implemented in this Application. All authorization logic is centralized in the middleware layer, with controllers focused solely on business logic.

**Status:** ✅ Fully Implemented

## Design Goals (Achieved)

1. ✅ **Centralize all authorization logic in middleware**
2. ✅ **Remove authorization checks from controllers**
3. ✅ **Provide clear, declarative authorization patterns**
4. ✅ **Maintain backward compatibility**
5. ✅ **Improve code maintainability**

## Implemented Authorization Patterns

### Pattern 1: Route-Based Authorization

Authorization requirements are defined in `routes.json`:

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
  },
  "/admin/users": {
    "controller": "AdminController",
    "method": "listUsers",
    "access": {
      "type": "admin_only"
    }
  }
}
```

**Access Types:**
- `public` - No authentication required
- `authenticated_only` - Must be logged in
- `admin_only` - Must have admin role
- `owner_or_admin` - Resource owner or admin can access
- `owner_only` - Only resource owner (no admin override)

### Pattern 2: AccessMiddleware Methods

The middleware provides methods for authorization:

```php
class AccessMiddleware {
    private function enforceRouteAccess(array $routeAccess, Identity $identity, array $request, string $accessUri): void
    private function applyResourceAuthorization(array $request, string $resourceType, string $idField): array
    private function loadResource(string $resourceType, int $resourceId): ?object
    private function getUserRole(?Identity $identity): string
}
```

### Pattern 3: Resource-Based Authorization

Resource authorization rules are configured in `routes.json` per-route:

```json
{
  "access": {
    "type": "owner_or_admin",
    "resource": "collections",      // Resource type to load
    "owner_field": "owner",         // Field to check for ownership
    "admin_override": true          // Allow admin access (default: true)
  }
}
```

Supported resource types:
- `collections` - Loaded from CollectionRepository
- `studies` - Loaded from StudiesRepository
- `schedules` - Loaded from ScheduleRepository
- `notifications` - Loaded from NotificationRepository

## Implementation (Complete)

### Phase 1: Enhanced AccessMiddleware ✅

- [x] Resource-based authorization via routes.json
- [x] Authorization enforcement methods
- [x] Route authorization enforcement
- [x] Request augmentation with auth context (`$request['authorized']`, `$request['authorized_resource']`)

### Phase 2: Route Definitions ✅

- [x] Authorization metadata in routes.json
- [x] Router passes auth requirements to middleware via routesRepository
- [x] Middleware checks authorization against routes.json configuration

### Phase 3: Controller Refactoring ✅

- [x] Removed AccessHelper::isOwnerOrAdmin() calls
- [x] Removed manual admin checks (`$isAdmin = ...`)
- [x] Removed manual authorization checks
- [x] Controllers use `$request['authorized_resource']` for pre-loaded resources
- [x] Controllers use `$request['targetUserId']` for user-scoped operations

### Phase 4: AccessHelper Status ✅

- [x] AccessHelper class retained as legacy utility (no longer used in controllers)
- [x] All controller imports updated
- [x] Documentation updated

## Current Implementation

### Controllers

```php
public function edit(array $request): array {
    // Trust: Authorization middleware already checked this user can access this resource
    $collection = $request['authorized_resource'];
    
    if (!$collection) {
        return ResponseBuilder::notFound('Collection not found');
    }
    
    // No authorization checks needed here - middleware handled it
    return ResponseBuilder::success([
        'collection' => $collection,
        'title' => 'Edit Collection'
    ], template: 'Collections/form');
}
```

### Middleware

```php
public function handle(array $request): array {
    // Get access configuration from routes.json
    $routeAccess = $this->routesRepository->getAccessForUri($accessUri, $params);
    
    // Enforce access based on route configuration
    if ($routeAccess !== null && $identity !== null) {
        $this->enforceRouteAccess($routeAccess, $identity, $request, $accessUri);
    }
    
    // Apply resource authorization for owner-based access
    if (isset($routeAccess['resource'])) {
        $request = $this->applyResourceAuthorization($request, $routeAccess['resource'], $routeAccess['owner_field'] ?? 'user_id');
    }
    
    return $request;
}
```

## Benefits (Realized)

### 1. Clean Separation of Concerns ✅
- Authorization logic in middleware layer
- Controllers focused on business logic
- Clear separation between auth and business rules

### 2. Consistent Authorization ✅
- Single source of truth (routes.json)
- Consistent error handling (403 exceptions)
- Consistent authorization patterns

### 3. Improved Maintainability ✅
- Easy to audit authorization rules (check routes.json)
- Easy to modify authorization logic (one place)
- Centralized authorization testing

### 4. Better Security ✅
- Hard to accidentally skip authorization checks
- Authorization enforced at framework level
- Consistent security model

### 5. Developer Experience ✅
- Clear documentation of authorization requirements
- Reduced boilerplate code
- Easier to understand authorization flow

## Testing

### Unit Tests
- `AccessMiddlewareTest` - Authorization enforcement
- `RoutesRepositoryTest` - Route access configuration
- Resource ownership validation

### Integration Tests
- End-to-end authorization flows
- Cross-controller authorization consistency
- Error handling consistency

### Security Tests
- Authorization bypass attempts
- Role escalation attempts
- Ownership bypass attempts

## Migration Complete

The migration from controller-based to middleware-based access control is **complete**.

### What Was Changed

1. **AccessMiddleware** - Enhanced to read from routes.json and enforce all authorization
2. **routes.json** - Added access configuration for all protected routes
3. **Controllers** - Removed all authorization checks, now trust middleware
4. **AccessHelper** - No longer used in controllers (retained as legacy utility)

### Current State

- All authorization centralized in middleware
- Controllers focus purely on business logic
- Configuration-driven access control via routes.json
- Consistent 403 responses for unauthorized access

---

**See Also:**
- [ACCESS_CONTROL_OVERVIEW.md](ACCESS_CONTROL_OVERVIEW.md) - Current access control architecture
- [ACCESS_CONTROL_IMPLEMENTATION.md](ACCESS_CONTROL_IMPLEMENTATION.md) - Implementation details
- [AUTHORIZATION_CURRENT_STATE.md](AUTHORIZATION_CURRENT_STATE.md) - Current authorization state

**Document Version:** 2.0  
**Last Updated:** 2025-02-08  
**Status:** ✅ Implemented and Active