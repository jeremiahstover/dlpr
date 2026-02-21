# Explicit Dependencies (No Magic)

To ensure the codebase remains maintainable, testable, and transparent, we follow a strict "No Magic" rule regarding dependencies.

## The "No Magic" Rule

All dependencies must be explicitly initialized and injected. The following are strictly prohibited:
- **No Service Locators**: Do not use a container to pull dependencies from inside a class.
- **No Static Globals**: Do not use static methods or global variables to access services or configuration.
- **No Hidden State**: Objects should not reach out to the environment to find what they need.
- **No Autowiring**: No dependency injection containers or automatic resolution.

## Composition Root (`App/App.php`)

All dependencies must be initialized in the **Composition Root**. This is the only place where objects are wired together.

In Application, `App/App.php` serves as the composition root where:
1. Configuration is loaded.
2. Clients (API wrappers) are initialized.
3. Repositories are initialized with their database connections.
4. Services are initialized with their required repositories and clients.
5. Controllers are initialized with their required services.

```php
// App/App.php
public function __construct() {
    // 1. Load config
    $this->config = $this->loadConfig();
    
    // 2. Initialize low-level dependencies (DB, Logger)
    $this->db = new Database($this->config['database']);
    
    // 3. Initialize API clients
    $this->verseClient = new VerseApiClient($this->config, $this->logger);
    $this->topicalClient = new TopicalApiClient($this->config, $this->logger);
    
    // 4. Initialize services with explicit dependencies
    $this->study = new StudyService(
        $studiesRepository,
        $this->userRepository,
        $this->scheduleClient,
        $this->verseClient,
        $this->config
    );
}
```

## Constructor Injection

All dependencies must be passed into a class via its constructor.

### ✅ CORRECT: Explicit dependency via constructor

```php
class StudyService {
    public function __construct(
        private StudyRepository $repository,
        private NotificationService $notifications
    ) {}
}

class StudiesController {
    public function __construct(
        private VerseApiClient $verseClient,
        private TopicalApiClient $topicalClient,
        private ScheduleApiClient $scheduleClient
    ) {}
    
    public function search(array $request) {
        return $this->verseClient->search($request['query']);
    }
}
```

### ❌ WRONG: Static access

```php
class StudyService {
    public function doSomething() {
        $data = StudyRepository::find(1); // Hidden dependency
    }
}
```

### ❌ WRONG: Service Locator / Container

```php
class StudyService {
    public function __construct(private Container $container) {}
    
    public function doSomething() {
        $repo = $this->container->get('StudyRepository'); // Hidden dependency
    }
}
```

## Controller Instantiation

Controllers are instantiated in `App::instantiateController` and receive their dependencies explicitly:

```php
// App/App.php
private function instantiateController(string $className): object {
    $fullClass = "Application\\App\\Routes\\Controllers\\{$className}";
    
    return match ($className) {
        'StudiesController' => new $fullClass(
            $this->verseClient,
            $this->topicalClient,
            $this->scheduleClient
        ),
        'UserController' => new $fullClass(
            $this->userRepository,
            $this->studyService
        ),
        default => new $fullClass()
    };
}
```

## Benefits

1. **Traceability**: You can follow the dependency flow from the top level (`App.php`) down to the lowest level by reading the code.
2. **Testability**: Dependencies can be easily mocked or stubbed in tests.
3. **Transparency**: You can see exactly what a class needs by looking at its constructor.
4. **Clarity**: Function signatures and constructors clearly state what a class needs to function.
5. **Decoupling**: Classes are not tied to a specific container or global state.
6. **No Hidden State**: All state is explicitly passed around, making it easier to reason about the application's behavior.
