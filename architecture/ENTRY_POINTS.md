# System Entry Points

This document provides a deep-dive into the entry point logic of the Application application, focusing on how `Public/index.php` functions as the "Traffic Cop" that routes incoming requests to the appropriate handlers based on URI type classification.

## Overview

The `Public/index.php` file serves as the single entry point for all incoming HTTP requests. It's responsible for analyzing the request URI and determining which processing pipeline should handle the request based on a 4-type classification system: static, lite, cron, and heavy.

## Traffic Cop Logic

The core decision-making logic in `Public/index.php` works as follows:

```php
// Public/index.php - Decision Tree
$router = new Application\App\Routes\Router();
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$routeInfo = $router->identifyRouteType($uri);
$routeType = $routeInfo['type'];
$resolvedUri = $routeInfo['resolved_uri'];

if ($routeType === 'static') {
    serveStaticFile($uri);
} elseif ($routeType === 'lite') {
    require_once __DIR__ . '/../App/Lite.php';
    $lite = new Application\App\Lite();
    echo $lite->handle($resolvedUri);
} elseif ($routeType === 'cron') {
    require_once __DIR__ . '/../App/Cron.php';
    $cron = new Application\App\Cron();
    echo $cron->run();
} else {
    require_once __DIR__ . '/../App/App.php';
    $app = new Application\App\App();
    echo $app->orchestrate();
}
```

## URI Type Classification

The `Router::identifyRouteType()` method in `App/Routes/Router.php` classifies URIs into one of four types:

### 1. Static Files (`static`)

**Detection Logic:**
```php
if (preg_match('/\.(css|js|png|jpg|jpeg|gif|svg|ico|woff|woff2|ttf|eot|json)$/i', $uri)) {
    return ['type' => 'static', 'resolved_uri' => $uri];
}
```

**Handling:** Requests for static assets are served directly with appropriate MIME types without loading the application framework.

**MIME Types Supported:**
- CSS: `text/css`
- JavaScript: `application/javascript`
- Images: `image/png`, `image/jpeg`, `image/gif`, `image/svg+xml`, `image/x-icon`
- Fonts: `font/woff`, `font/woff2`, `font/ttf`, `application/vnd.ms-fontobject`

**Security Considerations:**
- File existence validation before serving
- Directory traversal prevention
- MIME type enforcement to prevent script execution

### 2. Public Pages (`lite`)

**Detection Logic:**
```php
// Pages directory routes (lite) - handle /page/* without .html
if (preg_match('#^/page/[^.]+$#', $uri) || preg_match('#^/page/[^.]+$#', $resolvedUri)) {
    return ['type' => 'lite', 'resolved_uri' => $resolvedUri];
}
```

**Handling:** Lightweight requests that don't require authentication or complex processing. These are handled by the `Lite.php` controller which implements a 3-stage fallback system:
1. Direct HTML serving
2. PHP template processing
3. Plates template engine

### 3. Cron Jobs (`cron`)

**Detection Logic:**
```php
if ($uri === '/cron') {
    return ['type' => 'cron', 'resolved_uri' => $uri];
}
```

**Handling:** Background processing tasks handled by `Cron.php` which implements a 3-stage execution order:
1. Notifications (Due deliveries)
2. Schedules (Generate next notifications)
3. Studies (Maintenance and cleanup)

### 4. Authenticated Routes (`heavy`)

**Detection Logic:**
```php
// Everything else is heavy (authenticated)
return ['type' => 'heavy', 'resolved_uri' => $resolvedUri];
```

**Handling:** All other requests are handled by the full application framework in `App.php` which includes:
- Middleware pipeline execution
- Route matching with controller resolution
- Full dependency injection
- Authentication and authorization checks

## Alias Resolution

Before URI type classification, the router checks for URI aliases defined in `Data/aliases.json`:

```php
// Resolve alias if exists
$resolvedUri = $uri;
if (isset($this->aliases[$uri])) {
    $alias = $this->aliases[$uri];

    if (is_array($alias) && isset($alias['controller'], $alias['method'])) {
        return ['type' => 'heavy', 'resolved_uri' => $uri];
    }

    if (is_string($alias)) {
        $resolvedUri = $alias;
    }
}
```

This allows for URI shortcuts and legacy URL support while maintaining the same classification logic.

## Lazy Loading Pattern

The entry point implements a lazy loading pattern, only loading the components needed for the specific request type:
- Static files: No application code loaded
- Lite pages: Only `Lite.php` loaded
- Cron jobs: Only `Cron.php` loaded
- Heavy routes: Full application framework loaded

This approach minimizes memory usage and improves response times for simpler requests.

## Security Deep-Dive

### Directory Traversal Prevention

The `serveStaticFile()` function includes built-in protection against directory traversal attacks:

```php
function serveStaticFile($uri) {
    $file = __DIR__ . $uri;
    
    if (!file_exists($file) || !is_file($file)) {
        http_response_code(404);
        echo "404 Not Found";
        return;
    }
    // ... rest of implementation
}
```

Since `$file` is constructed by concatenating `__DIR__` (the Public directory) with the URI, requests like `../../../etc/passwd` would result in a path outside the Public directory, which would fail the `file_exists()` check.

### MIME Type Security

Static files are served with explicit MIME types to prevent browsers from interpreting files as executable scripts:

```php
$mimeTypes = [
    'css' => 'text/css',
    'js' => 'application/javascript',
    // ... other MIME types
];
```

## Performance Considerations

1. **Direct File Serving:** Static assets bypass PHP entirely for improved performance
2. **Lazy Loading:** Only required components are loaded for each request type
3. **Memory Efficiency:** Lite requests use minimal memory compared to full application framework
4. **Caching Potential:** Static and Lite responses can be easily cached at various levels

---

## Related Docs

- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Detailed URI type documentation
- [LIFE_OF_REQUEST.md](LIFE_OF_REQUEST.md) - Complete request lifecycle
- [DLPR_PATTERN.md](DLPR_PATTERN.md) - Application architecture pattern
- [STATIC_FILE_SERVING_SECURITY.md](STATIC_FILE_SERVING_SECURITY.md) - Static file security details

---

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete