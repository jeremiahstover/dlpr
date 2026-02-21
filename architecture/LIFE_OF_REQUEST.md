# Life of a Request

This document traces a complete request through the Application application, illustrating how an HTTP request is processed from entry point to response generation.

## Overview

Every HTTP request to the Application application follows a specific path through the system, determined by the URI type classification performed in `Public/index.php`. This document walks through that process with concrete examples.

## Example Trace: `/studies/123`

```
Browser Request: GET /studies/123
    ↓
Public/index.php (Entry Point)
    ├─ Parse URI: "/studies/123"
    ├─ Router.identifyRouteType() → 'heavy'
    ├─ Load App.php (lazy load)
    └─ App.orchestrate()
```

## Stage 1: Entry Point Processing

The request first hits `Public/index.php`, which determines the route type and loads the appropriate handler:

```php
$router = new Application\App\Routes\Router();
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$routeInfo = $router->identifyRouteType($uri);
$routeType = $routeInfo['type'];
$resolvedUri = $routeInfo['resolved_uri'];

// Since /studies/123 doesn't match static, lite, or cron patterns
// it defaults to 'heavy' type
require_once __DIR__ . '/../App/App.php';
$app = new Application\App\App();
echo $app->orchestrate();
```

## Stage 2: Application Orchestration

In `App.php`, the `orchestrate()` method coordinates the complete request processing pipeline:

```php
public function orchestrate(): string {
    try {
        // Parse the HTTP request into a standardized array
        $request = $this->parseRequest();
        
        // Execute the middleware stack
        $request = $this->executeMiddlewareStack($request);
        
        // Match the request to a controller/method
        $route = $this->router->match($request['uri'], $request);
        
        if (!$route) {
            $output = $this->handle404();
            return $this->handleContentNegotiation($output, $request);
        }
        
        // Instantiate and execute the controller
        $controller = $this->instantiateController($route['controller']);
        $output = $controller->{$route['method']}($request);
        
        // Handle content negotiation and return response
        return $this->handleContentNegotiation($output, $request);
        
    } catch (\Throwable $e) {
        return $this->handleException($e, $request);
    }
}
```

## Stage 3: Request Parsing

The `parseRequest()` method converts the raw HTTP request into a standardized array format:

```php
private function parseRequest(): array {
    $method = $_SERVER['REQUEST_METHOD'] ?? 'GET';
    $uri = parse_url($_SERVER['REQUEST_URI'] ?? '/', PHP_URL_PATH);
    
    $request = [
        'method' => $method,
        'uri' => $uri,
        'headers' => getallheaders() ?: [],
        'params' => [],
    ];
    
    // Parse query parameters
    parse_str(parse_url($_SERVER['REQUEST_URI'] ?? '', PHP_URL_QUERY) ?: '', $queryParams);
    $request['params'] = $queryParams;
    
    // Parse body for POST/PUT/PATCH
    if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
        $contentType = $_SERVER['CONTENT_TYPE'] ?? '';
        if (strpos($contentType, 'application/json') !== false) {
            $input = file_get_contents('php://input');
            $request['body'] = json_decode($input, true) ?: [];
        } else {
            $request['body'] = $_POST;
        }
    }
    
    return $request;
}
```

## Stage 4: Middleware Stack Execution

The request passes through a 7-stage middleware pipeline:

```php
private function executeMiddlewareStack(array &$request): array {
    $middleware = [
        new Logic\Middleware\ConfigMiddleware(),           // 1. Configuration loading
        $this->identityMiddleware,                          // 2. Identity establishment
        $this->accessMiddleware,                            // 3. Access control
        new Logic\Middleware\HttpMethodOverride(),          // 4. HTTP method override
        new Logic\Middleware\InputValidation(),             // 5. Input validation
        new Logic\Middleware\UserContext($this->userRepository), // 6. User context
        new Logic\Middleware\CsrfProtection($this->sessionRepository), // 7. CSRF protection
    ];
    
    foreach ($middleware as $m) {
        $request = $m->handle($request);
    }
    
    return $request;
}
```

Each middleware modifies the request array, adding data or performing validation checks.

## Stage 5: Route Matching

The router attempts to match the URI to a controller action:

```php
$route = $this->router->match($request['uri'], $request);

// Inside Router.php::match()
public function match(string $uri, array|null &$request = null): array|null {
    // Check for API prefix
    $isApi = false;
    if (strpos($uri, '/api/') === 0) {
        $isApi = true;
        $uri = substr($uri, 4); // Remove /api prefix
    }
    
    // Check if URI is an alias and resolve it
    if (isset($this->aliases[$uri])) {
        $alias = $this->aliases[$uri];
        
        if (is_array($alias) && isset($alias['controller'], $alias['method'])) {
            if ($request !== null && $isApi) {
                $request['is_api'] = true;
            }
            return $alias;
        }
        
        if (is_string($alias)) {
            $uri = $alias;
        }
    }
    
    // Look up route in main routes
    if (isset($this->routes[$uri])) {
        if ($request !== null && $isApi) {
            $request['is_api'] = true;
        }
        return $this->routes[$uri];
    }
    
    // Try pattern-based matching
    $matched = $this->matchPattern($uri, $request);
    if ($matched) {
        if ($request !== null && $isApi) {
            $request['is_api'] = true;
        }
        return $matched;
    }
    
    return null; // No route found
}
```

For our `/studies/123` example, assuming there's a route pattern like `/studies/{id}` defined in `Data/routes.json`, the router would:
1. Match the pattern
2. Extract the `id` parameter (123)
3. Return the controller configuration (likely `StudiesController` with `view` method)

## Stage 6: Controller Execution

The application instantiates and executes the matched controller:

```php
$controller = $this->instantiateController($route['controller']);
$output = $controller->{$route['method']}($request);
```

In our example, `StudiesController::view()` would be called:

```php
class StudiesController {
    public function view(array $request) {
        $studyId = $request['params']['id'] ?? null; // 123
        
        if (!$studyId) {
            return ResponseBuilder::error('Invalid ID');
        }
        
        // Business logic
        $studyData = $this->app->study->getStudy($studyId);
        
        if (!$studyData) {
            return ResponseBuilder::notFound('Study not found');
        }
        
        // Return data for presentation
        return ResponseBuilder::success([
            'study' => $studyData,
            'title' => $studyData['name']
        ], template: 'Studies/view');
    }
}
```

## Stage 7: Content Negotiation and Response

The controller's output is processed by `handleContentNegotiation()` to produce the final HTTP response:

```php
private function handleContentNegotiation(array $output, array $request): string {
    // Determine if client wants JSON
    $acceptHeader = $_SERVER['HTTP_ACCEPT'] ?? '';
    $isAjax = ($request['headers']['X-Requested-With'] ?? '') === 'xmlhttprequest';
    $wantJson = ($request['is_api'] ?? false) || 
                strpos($acceptHeader, 'application/json') !== false ||
                $isAjax;
    
    // Handle redirects
    if (isset($output['redirect'])) {
        header('Location: ' . $output['redirect'], true, $output['status'] ?? 302);
        return '';
    }
    
    // Handle error responses
    if (($output['error'] ?? false) || ($output['status'] ?? 200) >= 400) {
        http_response_code($output['status'] ?? 500);
        if ($wantJson) {
            header('Content-Type: application/json');
            return json_encode($output);
        }
        return $this->presenter->render('error', $output);
    }
    
    // Handle success responses
    if (isset($output['template'])) {
        $acceptsJson = strpos($acceptHeader, 'application/json') !== false;
        if ($acceptsJson || $isAjax) {
            header('Content-Type: application/json');
            return json_encode($output['data'] ?? $output);
        }
        return $this->presenter->render($output['template'], $output['data'] ?? $output);
    }
    
    // Fallback: return JSON if API, HTML otherwise
    if ($wantJson) {
        header('Content-Type: application/json');
        return json_encode($output);
    }
    return $this->presenter->render('default', $output);
}
```

## Exception Handling Flow

When exceptions occur at any stage, they are caught and processed:

```php
catch (\Throwable $e) {
    return $this->handleException($e, $request);
}

private function handleException(\Throwable $e, array $request): string {
    $statusCode = $e instanceof \RuntimeException && $e->getCode() >= 400 
        ? $e->getCode() 
        : 500;
    
    http_response_code($statusCode);
    
    $errorResponse = [
        'error' => true,
        'message' => $e->getMessage(),
        'code' => $statusCode
    ];
    
    // Return JSON for API requests, HTML for others
    if ($request['is_api'] ?? false) {
        header('Content-Type: application/json');
        return json_encode($errorResponse);
    }
    
    return $this->presenter->render('error', $errorResponse);
}
```

## Performance Considerations

1. **Lazy Loading:** Components are only loaded when needed
2. **Middleware Ordering:** Most expensive operations happen later in the pipeline
3. **Database Queries:** Deferred until controller execution
4. **Template Rendering:** Happens only after all business logic completes

## Memory Usage Throughout Pipeline

1. **Entry Point:** Minimal memory footprint
2. **Middleware Stack:** Gradually increases as context is built
3. **Controller Execution:** Peak memory usage during business logic
4. **Output Rendering:** Reduced memory as objects are converted to strings
5. **Response Sending:** Minimal memory as data is streamed to client

---

**Related Docs:**
- [ENTRY_POINTS.md](ENTRY_POINTS.md) - Entry point and routing overview
- [ARCHITECTURE_DEEP_DIVE.md](ARCHITECTURE_DEEP_DIVE.md) - Detailed architecture
- [API_NEGOTIATION_OVERVIEW.md](API_NEGOTIATION_OVERVIEW.md) - API/content negotiation details

**Document Version:** 2.0  
**Last Updated:** 2025-02-08  
**Status:** Accurate and Complete