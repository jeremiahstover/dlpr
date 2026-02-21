# Access Control Implementation

## Implementation Details

### AccessMiddleware Configuration

```php
class AccessMiddleware implements MiddlewareInterface
{
    public function __construct(
        private StudiesRepository $studiesRepository,
        private CollectionRepository $collectionRepository,
        private ScheduleRepository $scheduleRepository,
        private NotificationRepository $notificationRepository,
        private RoutesRepository $routesRepository,
        private ConfigManager $config
    ) {}

    public function handle(array $request): array
    {
        // Get user role from ConfigManager
        $role = $this->getUserRole($request['identity'] ?? null);

        // Get access configuration from routes.json
        $routeAccess = $this->routesRepository->getAccessForUri($uri);

        // Enforce access based on route configuration
        $this->enforceRouteAccess($routeAccess, $identity, $request, $uri);

        return $request;
    }
}
```

### ConfigManager Usage

All configuration values are retrieved through `ConfigManager` using dot-notation keys:

```php
// Roles
$role = $this->config->get('roles:admin', 'admin');      // 'admin'
$role = $this->config->get('roles:user', 'user');        // 'user'
$role = $this->config->get('roles:guest', 'guest');      // 'guest'

// Fields
$ownerField = $this->config->get('fields:user_id', 'user_id');
$ownerField = $this->config->get('fields:owner', 'owner');

// Resources
$resource = $this->config->get('resources:studies', 'studies');
$resource = $this->config->get('resources:collections', 'collections');
$resource = $this->config->get('resources:schedules', 'schedules');
$resource = $this->config->get('resources:notifications', 'notifications');

// Templates
Constants::TEMPLATE_COLLECTIONS_LIST = 'Collections/list';
Constants::TEMPLATE_USER_VIEW = 'User/view';
```

## Controller Patterns

### Resource Controller (Collections, Schedules, Notifications)

```php
public function edit(array $request): array {
    // Access control is already verified by middleware

    // Use pre-loaded resource from middleware
    $collection = $request['authorized_resource'] ?? null;
    if ($collection === null) {
        // Fallback if middleware didn't load (shouldn't happen in practice)
        $id = RequestHelper::getId($request);
        $collection = $this->collectionService->getCollection($id);
    }

    if (!$collection) {
        return ResponseBuilder::notFound('Collection not found');
    }

    // Business logic here
    $template = $this->config->get('templates:collections_form', 'Collections/form');
    return ResponseBuilder::success(
        data: ['collection' => $collection],
        template: $template
    );
}
```

### User-Scoped Controller (Profile, Settings)

```php
public function view(array $request): array {
    $currentUserId = (int)$request['identity']->id;

    // Use target user ID from middleware (handles ownership checking)
    $targetUserId = $request['targetUserId'] ?? $currentUserId;

    $user = $this->userService->getUserById($targetUserId);
    if ($user === null) {
        $msg = $this->config->get('messages:user_not_found', 'User not found');
        return ResponseBuilder::notFound($msg);
    }

    // Business logic here
    $template = $this->config->get('templates:user_view', 'User/view');
    return ResponseBuilder::success(
        data: ['user' => $user],
        template: $template
    );
}
```

### List Controller (Role-based Views)

```php
public function list(array $request): array {
    $config = $this->config;
    $userId = (int)($request['identity']->id ?? 0);
    $role = $request['identity']->role ?? $config->get('roles:guest', 'guest');
    $isAdmin = $role === $config->get('roles:admin', 'admin');

    switch ($role) {
        case $config->get('roles:admin', 'admin'):
            $items = $this->service->getAllItems();
            $template = $config->get('templates:items_list_admin', 'Items/list_admin');
            break;
        case $config->get('roles:user', 'user'):
            $items = $this->service->getItemsForUser($userId);
            $template = $config->get('templates:items_list', 'Items/list');
            break;
        default:
            return ResponseBuilder::forbidden('Invalid or missing role');
    }

    return ResponseBuilder::success(
        data: ['items' => $items],
        template: $template
    );
}
```

## Adding New Protected Routes

### Step 1: Define Route in routes.json

Add the route with access configuration:

```json
{
  "/collections/{id}/share": {
    "controller": "CollectionController",
    "method": "share",
    "access": {
      "type": "owner_or_admin",
      "resource": "collections",
      "owner_field": "owner"
    }
  }
}
```

Access types supported:
- `public` - No authentication required
- `authenticated_only` - Must be logged in
- `admin_only` - Must have admin role
- `owner_or_admin` - Resource owner or admin
- `owner_only` - Only resource owner (no admin override)

### Step 2: Implement Controller Method

```php
public function share(array $request): array {
    // Access already verified by middleware
    $collection = $request['authorized_resource'];

    // Business logic for sharing
    $this->collectionService->shareCollection($collection['id']);
    
    return ResponseBuilder::success(
        message: 'Collection shared successfully',
        template: Constants::TEMPLATE_COLLECTIONS_LIST
    );
}
```

### Step 4: Add Template (if needed)

Create template file in appropriate directory.

## Error Handling

### Authorization Errors

- **Middleware**: Throws `RuntimeException('Forbidden', 403)`
- **Controller**: Never checks authorization (trusts middleware)
- **ErrorHandler**: Converts exception to 403 response

### Not Found Errors

- **Controller**: Returns `ResponseBuilder::notFound()`
- **Middleware**: Doesn't interfere (404 vs 403 distinction)

### Validation Errors

- **InputValidation**: Sets `$request['errors']` and `$request['validated']`
- **Controller**: Uses validated data from middleware
- **No manual validation in controllers**

## Migration from Old Pattern

### Before (Controller-based Access)

```php
public function edit(array $request): array {
    $id = RequestHelper::getId($request);
    $collection = $this->collectionService->getCollection($id);
    
    // Access control in controller
    if ($collection['owner'] !== $request['identity']->id && 
        $request['identity']->role !== 'admin') {
        return ResponseBuilder::forbidden();
    }
    
    // Business logic
    return ResponseBuilder::success(['collection' => $collection]);
}
```

### After (Middleware-based Access)

```php
public function edit(array $request): array {
    // Access control already verified by middleware
    $collection = $request['authorized_resource'] ?? 
                 $this->collectionService->getCollection(RequestHelper::getId($request));
    
    if (!$collection) {
        return ResponseBuilder::notFound();
    }
    
    // Business logic
    return ResponseBuilder::success(['collection' => $collection]);
}
```

### Benefits of New Pattern

1. **Separation of Concerns**: Controllers focus on business logic
2. **Consistency**: All access rules in one place
3. **Maintainability**: Easy to audit and modify access policies
4. **Testability**: Separate testing of authorization and business logic
5. **Security**: Centralized, fail-closed access control
6. **Performance**: Resource loading cached in middleware

---

**See Also:**
- [ACCESS_CONTROL_OVERVIEW.md](ACCESS_CONTROL_OVERVIEW.md) - High-level concepts and principles
- [ACCESS_CONTROL_GUIDE.md](ACCESS_CONTROL_GUIDE.md) - Testing, migration, and troubleshooting

**Document Version:** 1.0  
**Document Version:** 1.1
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
