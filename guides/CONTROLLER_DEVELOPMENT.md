# Controller Development

Controllers handle HTTP requests, orchestrate business logic, and return responses. They are the entry point for application functionality after middleware processing.

## Overview

Controllers in ML2 follow these principles:
- **Single Responsibility**: Each controller handles one resource (Studies, Users, etc.)
- **Dependency Injection**: Services are injected via constructor
- **Request/Response**: Accept request arrays, return ResponseBuilder arrays
- **No Direct Output**: Controllers return data, never echo directly

## Controller Structure

### File Location

```
App/Routes/Controllers/
├── StudiesController.php
├── SchedulesController.php
├── IdentityController.php
├── ProfileController.php
└── ...
```

### Basic Controller Template

```php
<?php
declare(strict_types=1);

namespace MemorizeLive\App\Routes\Controllers;

use MemorizeLive\App\Logic\Helpers\ResponseBuilder;
use MemorizeLive\App\Logic\Services\MyService;
use MemorizeLive\App\Logic\Helpers\ConfigManager;

class MyController {
    
    public function __construct(
        private MyService $myService,
        private ConfigManager $config
    ) {}
    
    /**
     * List all items
     */
    public function list(array $request): array {
        $userId = (int)($request['identity']->id ?? 0);
        
        $items = $this->myService->getAll($userId);
        
        return ResponseBuilder::success(
            data: ['items' => $items],
            template: 'MyFeature/list'
        );
    }
    
    /**
     * Show create form
     */
    public function create(array $request): array {
        return ResponseBuilder::success(
            data: ['title' => 'Create Item'],
            template: 'MyFeature/form'
        );
    }
    
    /**
     * Store new item
     */
    public function store(array $request): array {
        $userId = (int)($request['identity']->id ?? 0);
        $data = $request['post'] ?? [];
        
        $result = $this->myService->create($userId, $data);
        
        if (!$result->isSuccess()) {
            return ResponseBuilder::error(
                message: $result->getError(),
                template: 'MyFeature/form'
            );
        }
        
        return ResponseBuilder::redirect(
            url: '/myfeature',
            message: 'Item created successfully'
        );
    }
}
```

## Request Array Structure

The `$request` array contains data from middleware:

```php
$request = [
    // HTTP data
    'uri' => '/studies',
    'method' => 'GET',
    'get' => ['page' => 1],
    'post' => ['title' => 'My Study'],
    'headers' => ['Accept' => 'text/html'],
    
    // Identity (set by IdentityMiddleware)
    'identity' => object {
        id: 123,
        email: 'user@example.com',
        role: 'user',
        interface: 2
    },
    
    // CSRF (set by CsrfProtection)
    'csrfToken' => 'abc123...',
    'includeCsrfToken' => true,
    
    // Config (set by ConfigMiddleware)
    'config' => [...],
    
    // Context (set by UserContext)
    'user_context' => [...],
    
    // Access control (set by AccessMiddleware)
    'studiesListScope' => 'own',
];
```

## Response Patterns

### HTML Response (Default)

```php
return ResponseBuilder::success(
    data: [
        'title' => 'Page Title',
        'items' => $items,
        'isAdmin' => true
    ],
    template: 'Studies/list'
);
```

### JSON Response (API)

```php
return ResponseBuilder::success(
    data: ['items' => $items],
    json: true
);
```

### Redirect

```php
return ResponseBuilder::redirect(
    url: '/studies',
    message: 'Study created successfully'
);
```

### Error Response

```php
return ResponseBuilder::error(
    message: 'Validation failed',
    errors: ['title' => 'Title is required'],
    template: 'Studies/form',
    status: 422
);
```

## Dependency Injection

### Register in App.php

Add your controller to the `instantiateController()` method:

```php
private function instantiateController(string $className): object {
    $fullClass = "MemorizeLive\\App\\Routes\\Controllers\\{$className}";
    
    return match ($className) {
        'MyController' => new $fullClass(
            $this->myService,
            $this->config
        ),
        // ... other controllers
    };
}
```

### Create Service

Add service creation method:

```php
private function createMyService(): MyService {
    return new MyService(
        $this->myRepository,
        $this->config
    );
}
```

### Register Service Getter

```php
public function getMyService(): MyService {
    if (!isset($this->services['my'])) {
        $this->services['my'] = $this->createMyService();
    }
    return $this->services['my'];
}
```

## Creating a New Controller

### Step 1: Create Controller File

```bash
touch App/Routes/Controllers/MyFeatureController.php
```

```php
<?php
declare(strict_types=1);

namespace MemorizeLive\App\Routes\Controllers;

use MemorizeLive\App\Logic\Helpers\ResponseBuilder;
use MemorizeLive\App\Logic\Services\MyFeatureService;
use MemorizeLive\App\Logic\Helpers\ConfigManager;

class MyFeatureController {
    
    public function __construct(
        private MyFeatureService $service,
        private ConfigManager $config
    ) {}
    
    public function index(array $request): array {
        return ResponseBuilder::success(
            data: ['title' => 'My Feature'],
            template: 'MyFeature/index'
        );
    }
}
```

### Step 2: Add Route

Edit `Data/routes.json`:

```json
{
    "/myfeature": {
        "controller": "MyFeatureController",
        "method": "index"
    }
}
```

### Step 3: Register in App.php

```php
private function instantiateController(string $className): object {
    // ...
    return match ($className) {
        'MyFeatureController' => new $fullClass(
            $this->getMyFeatureService(),
            $this->config
        ),
        // ...
    };
}
```

### Step 4: Create Template

Create `App/Presentation/MyFeature/index.php`:

```php
<?php $this->layout('layout', ['title' => $title]) ?>

<section class="section">
    <div class="container">
        <h1><?= $this->e($title) ?></h1>
    </div>
</section>
```

## Best Practices

### 1. Use Type Declarations

```php
public function list(array $request): array {
    // Always specify parameter and return types
}
```

### 2. Access Identity Safely

```php
$userId = (int)($request['identity']->id ?? 0);
$isAdmin = ($request['identity']->role ?? '') === 'admin';
```

### 3. Validate Input

```php
$data = $request['post'] ?? [];
$title = trim($data['title'] ?? '');

if (empty($title)) {
    return ResponseBuilder::error(
        message: 'Title is required',
        template: 'MyFeature/form'
    );
}
```

### 4. Use Config for Values

```php
$perPage = $this->config->get('pagination:per_page', 20);
```

### 5. Handle Service Results

```php
$result = $this->service->doSomething();

if (!$result->isSuccess()) {
    return ResponseBuilder::error(
        message: $result->getError(),
        template: 'MyFeature/form'
    );
}
```

### 6. Consistent Naming

| Action | Method Name | Route |
|--------|-------------|-------|
| List | `list()` | GET /resource |
| Show form | `create()` | GET /resource/create |
| Store | `store()` | POST /resource |
| Show | `show()` | GET /resource/{id} |
| Edit form | `edit()` | GET /resource/{id}/edit |
| Update | `update()` | PUT /resource/{id} |
| Delete | `destroy()` | DELETE /resource/{id} |

## Testing Controllers

### Unit Test Example

```php
<?php
require_once __DIR__ . '/../../vendor/autoload.php';

use MemorizeLive\App\Routes\Controllers\MyController;
use MemorizeLive\App\Logic\Services\MyService;
use MemorizeLive\App\Logic\Helpers\ConfigManager;

// Mock services
$mockService = new class extends MyService {
    public function getAll(int $userId): array {
        return [['id' => 1, 'name' => 'Test']];
    }
};

$mockConfig = new ConfigManager();

$controller = new MyController($mockService, $mockConfig);

// Test request
$request = [
    'identity' => (object)['id' => 123, 'role' => 'user'],
    'get' => []
];

$response = $controller->list($request);

// Assertions
assert($response['success'] === true);
assert(isset($response['data']['items']));
echo "✓ Test passed\n";
```

## Common Patterns

### Pagination

```php
public function list(array $request): array {
    $page = max(1, (int)($request['get']['page'] ?? 1));
    $perPage = $this->config->get('pagination:per_page', 20);
    
    $result = $this->service->getPaginated($page, $perPage);
    
    $pagination = PaginationHelper::create($request, $result['total'], $this->config);
    
    return ResponseBuilder::success(
        data: [
            'items' => $result['items'],
            'pagination' => $pagination
        ],
        template: 'MyFeature/list'
    );
}
```

### Form Handling

```php
public function store(array $request): array {
    $data = $request['post'] ?? [];
    
    // Validation
    $errors = [];
    if (empty(trim($data['title'] ?? ''))) {
        $errors['title'] = 'Title is required';
    }
    
    if (!empty($errors)) {
        return ResponseBuilder::error(
            message: 'Please fix the errors below',
            errors: $errors,
            template: 'MyFeature/form',
            data: ['input' => $data]
        );
    }
    
    // Process
    $result = $this->service->create($data);
    
    if (!$result->isSuccess()) {
        return ResponseBuilder::error(
            message: $result->getError(),
            template: 'MyFeature/form',
            data: ['input' => $data]
        );
    }
    
    return ResponseBuilder::redirect(
        url: '/myfeature',
        message: 'Created successfully'
    );
}
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
