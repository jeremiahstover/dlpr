# Middleware Stack Overview

The Application request processing pipeline uses a **7-tier middleware stack** that intercepts incoming requests and enriches them with context before they reach controllers. Each tier is responsible for a specific cross-cutting concern and executes in a strict order to ensure downstream middleware has access to upstream context.

## Execution Order

The middleware stack executes sequentially from Tier 1 through Tier 7:

| Tier | Middleware | Primary Responsibility | `$request` Key Added |
|------|-----------|----------------------|---------------------|
| 01 | [ConfigMiddleware](01_config.md) | Load global configuration | `$request['config']` |
| 02 | [IdentityMiddleware](02_identity.md) | Session-based authentication | `$request['identity']` |
| 03 | [AccessMiddleware](03_access.md) | Route-level authorization | `$request['access']` |
| 04 | [HttpMethodOverride](04_http_override.md) | HTTP verb normalization | `$request['original_method']` |
| 05 | [InputValidation](05_validation.md) | Input sanitization and casting | `$request['validated']` |
| 06 | [UserContext](06_context.md) | User preference hydration | `$request['user_context']` |
| 07 | [CsrfProtection](07_csrf.md) | CSRF token validation | `$request['csrf_valid']` |

## The Enrichment Pattern

Each middleware tier follows the **enrichment pattern**: it receives the `$request` array, adds specific keys, and passes the modified array to the next tier. This creates a cumulative context that controllers can rely on.

```php
// Initial request from router
$request = [
    'method' => 'POST',
    'uri' => '/studies/save',
    'query' => [],
    'post' => ['title' => 'Genesis 1'],
    'headers' => [],
];

// After Tier 01: ConfigMiddleware
$request['config'] = ['theme' => 'default', 'admins' => [...]];

// After Tier 02: IdentityMiddleware  
$request['identity'] = Identity { id: 42, email: 'user@example.com', role: 'user' };

// After Tier 03: AccessMiddleware
$request['access'] = ['role' => 'user'];
$request['authorized'] = true;

// After Tier 04: HttpMethodOverride
// No change (already POST)

// After Tier 05: InputValidation
$request['validated'] = ['title' => 'Genesis 1', 'type' => 'verse'];

// After Tier 06: UserContext
$request['user_context'] = ['timezone' => 'America/New_York', 'interface' => 2];

// After Tier 07: CsrfProtection
$request['csrf_valid'] = true;
$request['csrfToken'] = 'abc123...';
```

## Interface Contract

All middleware implement `MiddlewareInterface`:

```php
interface MiddlewareInterface {
    /**
     * Handle the request and pass it to the next middleware
     *
     * @param array $request The request array
     * @return array The modified request array
     * @throws \Exception Can throw exceptions to reject requests (401, 403, 400, etc.)
     */
    public function handle(array $request): array;
}
```

### Key Rules

- **Return `$request`**: Must return the modified request array
- **Throw for rejection**: Throw `RuntimeException` with HTTP codes (400, 401, 403) to reject requests
- **No business logic**: Middleware handles framework concerns only; business logic belongs in Services
- **No direct response output**: Middleware does not render templates or send responses

## Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HTTP Request                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 01: ConfigMiddleware                                          │
│  └─ Loads Data/config.json into $request['config']                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 02: IdentityMiddleware                                        │
│  └─ Validates session, creates Identity VO in $request['identity']  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 03: AccessMiddleware                                          │
│  └─ Reads routes.json, enforces access rules on $request['identity']│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 04: HttpMethodOverride                                        │
│  └─ Converts POST + _method=PUT to PUT in $request['method']        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 05: InputValidation                                           │
│  └─ Validates schemas, casts types to $request['validated']         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 06: UserContext                                               │
│  └─ Loads UserRepository, hydrates $request['user_context']         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 07: CsrfProtection                                            │
│  └─ Validates token for POST/PUT/DELETE, sets $request['csrf_valid']│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Router → Controller                          │
│  Controller receives fully enriched $request with all context       │
└─────────────────────────────────────────────────────────────────────┘
```

## Dependency Chain

The middleware stack has implicit dependencies based on execution order:

| Middleware | Depends On | Reason |
|-----------|-----------|--------|
| AccessMiddleware | IdentityMiddleware | Needs `$request['identity']` to check access |
| UserContext | IdentityMiddleware | Needs `$request['identity']->id` to load user |
| CsrfProtection | ConfigMiddleware | May need config for token settings |
| InputValidation | ConfigMiddleware | Needs `$request['config']` for schema initialization |

## Error Handling

Each tier throws specific exceptions that are caught by the central `ErrorHandler`:

| HTTP Code | Thrown By | Scenario |
|-----------|-----------|----------|
| 400 | InputValidation | Schema validation failure, missing required fields |
| 401 | IdentityMiddleware | Protected route accessed without valid session |
| 403 | AccessMiddleware | Insufficient permissions for resource |
| 403 | CsrfProtection | Invalid or missing CSRF token on state-changing request |

## Integration in App.php

The middleware stack is instantiated and executed in `App::executeMiddlewareStack()`:

```php
private function executeMiddlewareStack(array $request): array {
    $stack = new MiddlewareStack([
        new ConfigMiddleware(),
        new IdentityMiddleware($this->sessionRepository, $this->routesRepository, ...),
        new AccessMiddleware($this->routesRepository, $this->studiesRepository, ...),
        new HttpMethodOverride(),
        new InputValidation(),
        new UserContext($this->userRepository),
        new CsrfProtection($this->sessionRepository),
    ]);

    return $stack->execute($request);
}
```

The `MiddlewareStack` class iterates through each middleware in order:

```php
class MiddlewareStack {
    public function execute(array &$request): array {
        foreach ($this->middleware as $middleware) {
            $request = $middleware->handle($request);
        }
        return $request;
    }
}
```

## Controller Usage

Controllers receive the fully enriched request and access context through key access:

```php
public function save($app, array $request): array {
    // Access validated input (Tier 05)
    $title = $request['validated']['title'];
    
    // Access authenticated user (Tier 02)
    $userId = $request['identity']->id;
    
    // Access user preferences (Tier 06)
    $timezone = $request['user_context']['timezone'];
    
    // Access config values (Tier 01)
    $theme = $request['config']['theme'];
    
    // Business logic here...
}
```

## Related Documentation

- [Tier 01: ConfigMiddleware](01_config.md)
- [Tier 02: IdentityMiddleware](02_identity.md)
- [Tier 03: AccessMiddleware](03_access.md)
- [Tier 04: HttpMethodOverride](04_http_override.md)
- [Tier 05: InputValidation](05_validation.md)
- [Tier 06: UserContext](06_context.md)
- [Tier 07: CsrfProtection](07_csrf.md)
- [Access Control Checklist](ACCESS_CONTROL_CHECKLIST.md)
- [Access Control Standardization](ACCESS_CONTROL_STANDARDIZATION.md)
