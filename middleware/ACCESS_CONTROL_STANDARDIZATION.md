# Access Control Standardization

## Overview

All controllers now follow a consistent access control pattern that improves readability and maintainability.

## Standard Pattern

Every controller method that accesses a specific resource should follow this order:

```php
public function someMethod(mixed $app, array $request): array {
    $userId = (int)$request['identity']->id;

    // Extract admin status once at the beginning
    $isAdmin = isset($request['access']['role']) && $request['access']['role'] === 'admin';

    // 1. Load/validate resource exists (unconditional - must be first)
    $resource = $this->repository->getById($id);
    if (!$resource) {
        return $this->responseBuilder->notFound('Resource not found');
    }

    // 2. Check access (universal - everyone goes through this)
    if (!AccessHelper::isOwnerOrAdmin($resource, $userId, $isAdmin, 'owner_field')) {
        return $this->responseBuilder->forbidden('Access denied');
    }

    // 3. Determine user's role for this resource (extract data)
    $isOwner = (int)$resource['owner_field'] === $userId;

    // 4. Proceed with role-specific logic (now conditions are clear)
    if ($isOwner) {
        // Owner-specific logic
    }
    if ($isAdmin) {
        // Admin-specific logic
    }

    // Continue with business logic...
}
```

## Key Principles

### 1. Unconditional Resource Loading
- Resource existence check is **always first**
- Not conditional on user role
- Resource must exist before any access checks

### 2. Universal Access Check
- Access check happens **after** resource is loaded
- **Always performed** (not conditional on role)
- Uses `AccessHelper::isOwnerOrAdmin()` for consistency
- Either the user passes (owner or admin) or gets 403

### 3. Clear Role Determination
- Determine user's role **after** access check
- Extract data from resource and request
- Keep it simple and explicit

### 4. Role-Specific Logic Last
- Only execute role-specific logic **after** access is verified
- Clear separation between access check and business logic
- No nesting access checks inside role branches

## Changes Made

### CollectionController::save()

**Before (muddled and backwards):**
```php
$isAdmin = isset($request['access']['role']) && $request['access']['role'] === 'admin';

$owner = $userId;
if ($isAdmin && isset($post['owner']) && is_numeric($post['owner'])) {
    $owner = (int)$post['owner'];
}

if ($id) {
    $existing = $this->collectionService->getCollection($id);
    if (!$existing) {
        return $this->responseBuilder->notFound('Collection not found');
    }
    if (!AccessHelper::isOwnerOrAdmin($existing, $userId, $isAdmin, 'owner')) {
        return $this->responseBuilder->forbidden('Access denied');
    }
    $owner = (int)$existing['owner'];  // ← Overwritten after access check!
}
```

**After (clear and follows pattern):**
```php
$isAdmin = isset($request['access']['role']) && $request['access']['role'] === 'admin';

if ($id) {
    // 1. Load resource (unconditional)
    $existing = $this->collectionService->getCollection($id);
    if (!$existing) {
        return $this->responseBuilder->notFound('Collection not found');
    }

    // 2. Check access (universal - owner or admin)
    if (!AccessHelper::isOwnerOrAdmin($existing, $userId, $isAdmin, 'owner')) {
        return $this->responseBuilder->forbidden('Access denied');
    }

    // 3. Determine owner (extract data - preserve existing owner)
    $owner = (int)$existing['owner'];
} else {
    // Creating new collection - determine owner based on admin status
    $owner = $userId;
    if ($isAdmin && isset($post['owner']) && is_numeric($post['owner'])) {
        $owner = (int)$post['owner'];
    }
}
```

### Controllers Already Following Pattern

These controllers were already following the correct pattern:

- **SchedulesController**: All methods (view, edit, updateSchedule, incrementPosition, delete)
- **NotificationsController**: All methods (view, edit, updateNotification, delete)
- **StudiesController**: All methods (view, edit, save, delete, incrementPosition, reset)
  - Note: StudiesController uses owner-only access (no admin override), which is intentional

## Benefits

1. **Consistency**: All controllers follow the same pattern
2. **Readability**: Code reads clearly: load → check → determine → proceed
3. **Security**: Access checks are universal and never skipped
4. **Maintainability**: Changes to access logic are easier with clear structure
5. **Debugging**: Easier to trace execution flow with linear progression

## AccessHelper Usage

The `AccessHelper::isOwnerOrAdmin()` method centralizes owner-or-admin checks:

```php
use MemorizeLive\App\Logic\Helpers\AccessHelper;

// Default owner field: 'user_id'
AccessHelper::isOwnerOrAdmin($resource, $userId, $isAdmin);

// Custom owner field: 'owner' (for collections)
AccessHelper::isOwnerOrAdmin($resource, $userId, $isAdmin, 'owner');
```

See [Access Architecture](../architecture/ACCESS_CONTROL_ARCHITECTURE.md) and [Authorization Patterns](../architecture/AUTHORIZATION_PATTERNS.md) for full details.

## Testing

Comprehensive test suite in `Tests/test-access-control-standardization.php` verifies:

- Owner can access their own resources
- Admin can access any resource
- Non-owner/non-admin gets denied
- Custom owner fields work correctly
- Type safety and edge cases

All tests pass successfully.

## Migration Guide

When adding new controller methods:

1. **Extract admin status** once at the beginning
2. **Load resource** unconditionally (return 404 if not found)
3. **Check access** universally using `AccessHelper::isOwnerOrAdmin()` (return 403 if denied)
4. **Determine role** after access is verified
5. **Execute role-specific logic** last

Never:
- ❌ Nest access checks inside role-specific branches
- ❌ Make resource loading conditional on role
- ❌ Check access only in specific role branches
- ❌ Overwrite data after access checks

## Related Documentation

- [Access Control Architecture](../architecture/ACCESS_CONTROL_ARCHITECTURE.md) - Access control details
- [Authorization Patterns](../architecture/AUTHORIZATION_PATTERNS.md) - Authorization patterns
- [ARCHITECTURE_OVERVIEW.md](../ARCHITECTURE_OVERVIEW.md) - Overall architecture
