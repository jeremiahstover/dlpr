# Tier 03: AccessMiddleware

**Purpose**: Enforce access control rules defined in `routes.json` for route-level and resource-level authorization.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['access']` | `array` | Access metadata including `role` |
| `$request['access_uri']` | `string` | URI with `/api/` prefix stripped for matching |
| `$request['authorized']` | `bool` | Set to `true` when all access checks pass |
| `$request['targetUserId']` | `int` | For user routes, the target user ID |
| `$request['targetStudyId']` | `int` | For study routes, the target study ID |
| `$request['study']` | `array` | Pre-loaded study data for study routes |
| `$request['studiesListScope']` | `string` | `'all'` for admin, `'own'` for users |
| `$request['authorized_resource']` | `array` | Pre-loaded resource data for ownership routes |
| `$request['authorized_resource_type']` | `string` | Type of authorized resource |

## Responsibilities

1. **Route Access Enforcement**: Reads access rules from `routes.json` and applies them
2. **Role-Based Access**: Enforces `admin_only`, `authenticated_only`, `public` access types
3. **Ownership Verification**: Checks resource ownership for `owner_only` and `owner_or_admin` routes
4. **Resource Loading**: Loads study/collection/schedule data to verify ownership
5. **Admin Bypass**: Allows admin users to access any owned resource

## Access Types

The `routes.json` file defines access rules with these types:

| Access Type | Description | Example Route |
|-------------|-------------|---------------|
| `public` | No authentication required | `/validate-passage` |
| `authenticated_only` | Must be logged in, no ownership check | `/studies/create` |
| `admin_only` | Must have admin role | `/admin/dashboard` |
| `owner_only` | Must be resource owner (no admin bypass) | `/studies/{id}/edit` |
| `owner_or_admin` | Must be owner OR have admin role | `/studies/{id}/reset` |

## routes.json Access Configuration

```json
{
    "/studies/{id}/edit": {
        "controller": "StudiesController",
        "method": "edit",
        "access": {
            "type": "owner_only",
            "resource": "studies",
            "owner_field": "user_id"
        }
    },
    "/studies/{id}/reset": {
        "controller": "StudiesController",
        "method": "reset",
        "access": {
            "type": "owner_or_admin",
            "resource": "studies",
            "owner_field": "user_id"
        }
    }
}
```

### Access Configuration Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | One of the access types listed above |
| `resource` | For ownership types | Resource name for loading (`studies`, `collections`, etc.) |
| `owner_field` | For ownership types | Database field containing owner ID |
| `ownership` | Optional | Additional ownership constraint for user routes |

## Ownership Verification Flow

```
Request with identity
       │
       ▼
┌────────────────────────┐
│ Is access type         │─No──► Skip to next check
│ owner_only or          │
│ owner_or_admin?        │
└────────────────────────┘
       │ Yes
       ▼
┌────────────────────────┐
│ Is user admin?         │─Yes──► Grant access (if owner_or_admin)
└────────────────────────┘
       │ No
       ▼
┌────────────────────────┐
│ Extract resource ID    │
│ from URI or params     │
└────────────────────────┘
       │
       ▼
┌────────────────────────┐
│ Load resource from     │
│ repository             │
└────────────────────────┘
       │
       ▼
┌────────────────────────┐
│ Compare owner_field    │
│ with identity->id      │
└────────────────────────┘
       │
       ▼
┌────────────────────────┐
│ Match?                 │─No──► Throw 403 Forbidden
└────────────────────────┘
       │ Yes
       ▼
   Grant access
```

## Resource Loading

The middleware loads resources from appropriate repositories based on the `resource` field:

```php
private function loadResource(string $resourceType, int $id): ?array {
    return match($resourceType) {
        'collections' => $this->collectionRepository?->findById($id),
        'schedules' => $this->scheduleRepository?->find($id),
        'notifications' => $this->notificationRepository?->find($id),
        'studies' => $this->studiesRepository?->getStudy($id),
        default => null,
    };
}
```

## User Route Ownership

For routes like `/user/{id}/*`, the middleware extracts the target user ID and enforces ownership:

```php
private function handleUserOwnership(Identity $identity, array &$request, string $uri): void {
    $targetUserId = $this->extractTargetUserIdFromRequest($request);
    
    if ($targetUserId !== (int)$identity->id && $identity->role !== 'admin') {
        throw new \RuntimeException('Forbidden', 403);
    }
    
    $request['targetUserId'] = $targetUserId;
}
```

## Studies Context

The middleware pre-loads study data for study routes to avoid duplicate queries in controllers:

```php
private function applyStudiesAccessContext(array &$request, ?Identity $identity, string $uri): void {
    // Set list scope for /studies route
    if ($uri === '/studies') {
        $request['studiesListScope'] = $identity->role === 'admin' ? 'all' : 'own';
        return;
    }
    
    // Extract and load study for study routes
    $studyId = $this->extractStudyIdFromUri($uri);
    if ($studyId !== null && $this->studiesRepository !== null) {
        $request['study'] = $this->studiesRepository->getStudy($studyId);
        $request['targetStudyId'] = $studyId;
    }
}
```

## API Prefix Stripping

The middleware strips `/api/` prefixes to match routes configured without the prefix:

```php
private function stripApiPrefix(string $uri): string {
    if (str_starts_with($uri, '/api/')) {
        return '/' . ltrim(substr($uri, 4), '/');
    }
    return $uri;
}
```

## Error Handling

| Exception | Code | Condition |
|-----------|------|-----------|
| `RuntimeException` | 403 | Insufficient permissions for resource |
| `RuntimeException` | 403 | CSRF validation failure |

## Downstream Usage

Controllers rely on access middleware to pre-load resources and validate ownership:

```php
public function edit($app, array $request): array {
    // Study already loaded by AccessMiddleware
    $study = $request['study'];
    
    // Ownership already verified, safe to proceed
    return [
        'study' => $study,
        // ...
    ];
}
```

## Dependencies

- **RoutesRepository**: Reads access rules from `routes.json`
- **StudiesRepository**: Loads study data for ownership checks
- **CollectionRepository**: Loads collection data
- **ScheduleRepository**: Loads schedule data
- **NotificationRepository**: Loads notification data
- **ConfigManager**: Reads resource and role configuration
- **Identity**: Reads `$request['identity']` from upstream

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 02: IdentityMiddleware](02_identity.md)
- [Tier 04: HttpMethodOverride](04_http_override.md)
- [Access Control Checklist](ACCESS_CONTROL_CHECKLIST.md)
- [Access Control Standardization](ACCESS_CONTROL_STANDARDIZATION.md)
