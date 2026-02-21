# Testing Guide

ML2 uses a custom testing framework with PHP scripts. Tests are organized by type and can be run individually or in groups.

## Overview

The testing approach follows these principles:
- **No External Dependencies**: Pure PHP, no PHPUnit required
- **Organized by Type**: Unit, Integration, Feature, Acceptance, EndToEnd
- **In-Memory Testing**: SQLite for database tests
- **Simple Assertions**: Built-in assert functions

## Test Organization

```
Tests/
├── Unit/                    # Isolated component tests
│   ├── test-routes-json-as-source-of-truth.php
│   ├── test-config-defaults-base-url-encryption-key.php
│   └── run-tests.php         # Run all unit tests
├── Integration/             # Component interaction tests
│   ├── test-router-enhancements.php
│   └── README.md
├── Feature/                 # Feature-specific tests
├── Acceptance/              # User story tests
├── EndToEnd/                # Full workflow tests
├── Security/                # Security-focused tests
├── API/                     # API endpoint tests
└── TestHelper.php          # Shared test utilities
```

## TestHelper

Provides common utilities for tests:

```php
require_once __DIR__ . '/../TestHelper.php';

// Create in-memory database
$pdo = TestHelper::createInMemoryDatabase();

// Create file-based test database
$pdo = TestHelper::createTestDatabase('/tmp/test.db');

// Manage sessions
TestHelper::clearSession();
TestHelper::startFreshSession();
```

## Writing Tests

### Basic Test Structure

```php
<?php
declare(strict_types=1);

require_once __DIR__ . '/../TestHelper.php';

class MyFeatureTest {
    private int $passed = 0;
    private int $failed = 0;
    
    public function run(): void {
        echo "=== Testing My Feature ===\n\n";
        
        $this->testSomethingWorks();
        $this->testSomethingElse();
        
        echo "\n=== Summary ===\n";
        echo "Passed: {$this->passed}\n";
        echo "Failed: {$this->failed}\n";
        
        exit($this->failed > 0 ? 1 : 0);
    }
    
    private function assert(bool $condition, string $message): void {
        if ($condition) {
            $this->passed++;
            echo "✓ {$message}\n";
        } else {
            $this->failed++;
            echo "✗ {$message}\n";
        }
    }
    
    private function testSomethingWorks(): void {
        $result = someFunction();
        $this->assert($result === true, 'Function returns true');
    }
}

// Run the test
(new MyFeatureTest())->run();
```

### Unit Test Example

```php
<?php
declare(strict_types=1);

use MemorizeLive\App\Data\Repositories\UserRepository;

require_once __DIR__ . '/../TestHelper.php';
require_once __DIR__ . '/../../App/Data/Repositories/UserRepository.php';

class UserRepositoryTest {
    private \PDO $pdo;
    private UserRepository $repo;
    private int $passed = 0;
    private int $failed = 0;
    
    public function __construct() {
        $this->pdo = TestHelper::createInMemoryDatabase();
        $this->repo = new UserRepository($this->pdo);
    }
    
    public function run(): void {
        echo "=== UserRepository Tests ===\n\n";
        
        $this->testCreateUser();
        $this->testFindById();
        $this->testFindByEmail();
        $this->testUpdateUser();
        $this->testDeleteUser();
        
        echo "\n=== Summary ===\n";
        echo "Passed: {$this->passed}\n";
        echo "Failed: {$this->failed}\n";
        
        exit($this->failed > 0 ? 1 : 0);
    }
    
    private function assert(bool $condition, string $message): void {
        if ($condition) {
            $this->passed++;
            echo "✓ {$message}\n";
        } else {
            $this->failed++;
            echo "✗ {$message}\n";
        }
    }
    
    private function testCreateUser(): void {
        $id = $this->repo->create([
            'email' => 'test@example.com',
            'name' => 'Test User'
        ]);
        
        $this->assert($id > 0, 'Create user returns valid ID');
    }
    
    private function testFindById(): void {
        $id = $this->repo->create(['email' => 'find@example.com', 'name' => 'Find']);
        $user = $this->repo->findById($id);
        
        $this->assert($user !== null, 'Find by ID returns user');
        $this->assert($user['email'] === 'find@example.com', 'User has correct email');
    }
    
    private function testFindByEmail(): void {
        $this->repo->create(['email' => 'email@example.com', 'name' => 'Email']);
        $user = $this->repo->findByEmail('email@example.com');
        
        $this->assert($user !== null, 'Find by email returns user');
    }
    
    private function testUpdateUser(): void {
        $id = $this->repo->create(['email' => 'update@example.com', 'name' => 'Old']);
        $this->repo->update($id, ['name' => 'New']);
        $user = $this->repo->findById($id);
        
        $this->assert($user['name'] === 'New', 'Update changes name');
    }
    
    private function testDeleteUser(): void {
        $id = $this->repo->create(['email' => 'delete@example.com', 'name' => 'Delete']);
        $this->repo->delete($id);
        $user = $this->repo->findById($id);
        
        $this->assert($user === null, 'Delete removes user');
    }
}

(new UserRepositoryTest())->run();
```

## Running Tests

### Run Single Test

```bash
php Tests/Unit/test-routes-json-as-source-of-truth.php
```

### Run All Unit Tests

```bash
php Tests/Unit/run-tests.php
```

### Run Specific Category

```bash
# Integration tests
php Tests/Integration/test-router-enhancements.php

# Feature tests
php Tests/Feature/test-some-feature.php
```

## Test Categories

### Unit Tests

Test individual components in isolation:

- Repositories
- Services
- Helpers
- Middleware

### Integration Tests

Test component interactions:

- Router + Middleware
- Controller + Service
- Service + Repository

### Feature Tests

Test complete features:

- User registration flow
- Study creation
- Notification delivery

### Acceptance Tests

Test user stories:

- "As a user, I can create a study"
- "As an admin, I can view all studies"

### End-to-End Tests

Test full workflows:

- Register → Login → Create Study → Schedule → Notification

## Testing Patterns

### Database Testing

```php
// Use in-memory database
$pdo = TestHelper::createInMemoryDatabase();

// Or use test file
$pdo = TestHelper::createTestDatabase('/tmp/test.db');

// Always clean up
if (file_exists('/tmp/test.db')) {
    unlink('/tmp/test.db');
}
```

### Mocking Services

```php
// Create mock service
$mockService = new class extends MyService {
    public function expensiveOperation(): array {
        return ['mocked' => true];
    }
};

// Inject mock
$controller = new MyController($mockService);
```

### Testing Controllers

```php
// Build request array
$request = [
    'identity' => (object)[
        'id' => 123,
        'email' => 'test@example.com',
        'role' => 'user'
    ],
    'get' => ['page' => 1],
    'post' => ['title' => 'Test Study']
];

// Call controller
$response = $controller->list($request);

// Assert response
assert($response['success'] === true);
assert(isset($response['data']['items']));
```

### Testing Middleware

```php
$middleware = new IdentityMiddleware($userRepo);

$request = ['uri' => '/studies', 'headers' => []];
$result = $middleware->process($request);

// Assert middleware added identity
assert(isset($result['identity']));
```

## Best Practices

1. **One Test Class Per Component**: `UserRepositoryTest`, `StudyServiceTest`
2. **Descriptive Test Names**: `testCreateUserReturnsValidId`
3. **Independent Tests**: Each test should set up its own data
4. **Clean Up**: Remove test databases and files
5. **Exit Codes**: Return 0 for success, 1 for failure
6. **Clear Output**: Use ✓ for pass, ✗ for fail

## Common Assertions

```php
// Equality
$this->assert($actual === $expected, 'Values match');

// Null check
$this->assert($result === null, 'Returns null when not found');

// Array has key
$this->assert(isset($array['key']), 'Array has key');

// String contains
$this->assert(strpos($haystack, $needle) !== false, 'String contains');

// Exception thrown
try {
    $service->doSomethingInvalid();
    $this->assert(false, 'Should throw exception');
} catch (\InvalidArgumentException $e) {
    $this->assert(true, 'Throws correct exception');
}

// Type check
$this->assert(is_array($result), 'Returns array');
$this->assert($result instanceof User, 'Returns User object');
```

## Debugging Failed Tests

```php
// Add debug output
private function testSomething(): void {
    $result = $this->repo->findById(999);
    
    // Debug: print actual value
    echo "Debug: result = " . var_export($result, true) . "\n";
    
    $this->assert($result === null, 'Returns null for invalid ID');
}
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
