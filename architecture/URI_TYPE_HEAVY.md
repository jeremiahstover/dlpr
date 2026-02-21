# URI Type: Heavy Routes

## Classification Logic

Heavy routes are the default classification for any URI that doesn't match the static, lite, or cron patterns:

```php
// Everything else is heavy (authenticated)
return ['type' => 'heavy', 'resolved_uri' => $resolvedUri];
```

## Processing Pipeline

Heavy routes are processed by the full application framework in `App/App.php`:

```php
public function orchestrate(): void {
    try {
        // Parse the HTTP request into a standardized array
        $request = $this->parseRequest();
        
        // Execute the middleware stack
        $request = $this->executeMiddlewareStack($request);
        
        // Match the request to a controller/method
        $routeConfig = $this->router->match($request['uri'], $request);
        
        // Instantiate and execute the controller
        $output = $this->executeController($routeConfig, $request);
        
        // Render the response
        $response = $this->renderOutput($output, $request);
        
        // Send the HTTP response
        $this->sendResponse($response);
    } catch (Exception $e) {
        $this->handleException($e);
    }
}
```

## Characteristics

- **Full Middleware Stack**: All 7 middleware components execute
- **Authentication Required**: User identity established and validated
- **Complex Business Logic**: Database queries, service layer operations
- **Template Rendering**: Full presentation layer with Plates engine

## Middleware Stack

1. **ConfigMiddleware**: Configuration loading
2. **IdentityMiddleware**: User authentication
3. **AccessMiddleware**: Authorization checks
4. **HttpOverrideMiddleware**: Method override handling
5. **InputValidationMiddleware**: Input validation
6. **ContextMiddleware**: Request context setup
7. **CsrfMiddleware**: CSRF protection

---

**See Also:**
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Overview of all URI types
- [LIFE_OF_REQUEST.md](LIFE_OF_REQUEST.md) - Detailed request lifecycle

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete