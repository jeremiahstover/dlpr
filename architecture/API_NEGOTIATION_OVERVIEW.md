# API Content Negotiation Overview

## Overview

The Application application implements content negotiation to serve different response formats based on the request type. API requests are identified by the `/api/` prefix and return JSON responses, while standard web requests return HTML responses rendered through the template system.

## API Detection Logic

API requests are detected in the router's `match()` method:

```php
// Router.php::match() - API detection
if (strpos($uri, '/api/') === 0) {
    $isApi = true;
    $uri = str_replace('/api/', '/', $uri); // Strip /api prefix
    $request['is_api'] = true; // Flag for controller
}
```

### Key Steps

1. **Prefix Check**: Identify URIs that begin with `/api/`
2. **Flag Setting**: Set the `is_api` flag in both local scope and request data
3. **URI Normalization**: Strip the `/api/` prefix to normalize the URI for route matching
4. **Continued Processing**: Proceed with normal route matching using the normalized URI

## URI Transformation Example

```
Original Request: GET /api/studies/123
After Processing: GET /studies/123 (with is_api=true flag)
```

This approach allows the same controller logic to handle both API and web requests while enabling different response formatting.

## Content Negotiation Implementation

The content negotiation happens in `App.php::handleContentNegotiation()` and is more sophisticated than a simple API check:

```php
private function handleContentNegotiation(array $output, array $request): string {
    // Determine requested format from multiple sources
    $acceptHeader = $request['headers']['Accept'] ?? 'text/html';
    $acceptsJson = strpos($acceptHeader, 'application/json') !== false;
    $requestedWith = $request['headers']['X-Requested-With'] ?? '';
    $isAjax = strtolower((string)$requestedWith) === 'xmlhttprequest';
    $isApi = isset($request['is_api']) && $request['is_api'] === true;

    // JSON is returned if: API prefix OR Accept: application/json OR AJAX request
    $wantJson = $acceptsJson || $isApi;

    // Set HTTP status
    $status = $output['status'] ?? 200;
    http_response_code($status);

    // Handle redirects
    if ($status === 302 && isset($output['redirectUrl'])) {
        header("Location: {$output['redirectUrl']}", true, $status);
        return '';
    }

    // Error responses
    if (isset($output['error']) && $output['error'] === true) {
        $template = $output['template'] ?? 'error-show';
        $data = $output['data'] ?? $output;

        return $wantJson
            ? json_encode(['error' => true, 'status' => $status, 'message' => $data['message']])
            : $this->presenter->render($template, $data);
    }

    // Success responses with template
    if (isset($output['template'])) {
        $data = $output['data'] ?? $output;

        // If JSON explicitly requested (fetch/XHR), return JSON even with template
        if ($acceptsJson || $isAjax) {
            return json_encode($output);
        }

        // Otherwise render HTML
        return $this->presenter->render($output['template'], $data);
    }

    // Fallback for responses without template
    return $wantJson ? json_encode($output) : $this->presenter->render(null, $output);
}
```

### Response Format Detection

The system detects JSON requests through multiple mechanisms:

1. **API Prefix**: URI starts with `/api/` → `$request['is_api'] = true`
2. **Accept Header**: `Accept: application/json` → `$acceptsJson = true`
3. **AJAX Requests**: `X-Requested-With: XMLHttpRequest` → `$isAjax = true`

### Response Formats

1. **API/AJAX/JSON Requests**: JSON formatted data
2. **Web Requests**: HTML rendered through the template engine
3. **Mixed Mode**: Even `/api/*` endpoints can return HTML if they provide a template and JSON isn't explicitly requested

## Request Format Differences

### Web Requests

Web requests typically use:
- **Form Data**: Standard HTML form submissions
- **Query Parameters**: URL parameters for filtering/sorting
- **Session-Based Authentication**: User identity maintained through sessions
- **HTML Responses**: Rich markup with styling and layout

### API Requests

API requests typically use:
- **JSON Body**: Structured data payloads
- **Query Parameters**: URL parameters for filtering/sorting
- **Token-Based Authentication**: Bearer tokens or API keys
- **JSON Responses**: Machine-readable structured data

## Parameter Handling Differences

While the core routing logic remains the same, API and web requests handle parameters differently:

### Path Parameters

Both request types handle path parameters identically:
```php
// Route pattern: /studies/{id}
// Request: /api/studies/123 or /studies/123
// Result: $request['params']['id'] = 123
```

### Query Parameters

Query parameters are handled consistently:
```php
// Request: /api/studies?status=active&limit=10 or /studies?status=active&limit=10
// Result: $request['params']['status'] = 'active', $request['params']['limit'] = 10
```

### Request Body

Request body handling is managed through the framework's standard input processing. Both web form submissions and JSON payloads are normalized into the `$request['params']` array by the middleware stack before reaching controllers.

Controllers access request data uniformly:

```php
// Both web forms and API requests populate $request['params']
$name = $request['params']['name'] ?? null;
$data = $request['params']; // All input parameters
```

The `InputValidationMiddleware` handles validation regardless of the request source.

## Authentication Differences

### Web Authentication

Web requests typically use session-based authentication:
- **Session Cookies**: Automatic session management
- **Form Login**: Username/password submission
- **CSRF Protection**: Cross-site request forgery tokens

### API Authentication

API requests can use token-based authentication via the Authorization header:
- **Authorization Header**: `Authorization: Bearer <token>` (stub implementation)
- **Session Fallback**: If no valid token, falls back to session authentication
- **Stateless Ready**: Architecture supports stateless auth (implementation pending)

**Note**: The `authenticateViaHeader()` method in `IdentityMiddleware` is currently a stub that returns `null`. Full bearer token/API key validation is planned for future implementation. All API requests currently rely on session-based authentication.

## Response Format Differences

### JSON Responses (API)

```json
{
    "status": "success",
    "data": {
        "id": 123,
        "title": "Sample Study",
        "created_at": "2023-01-01T12:00:00Z"
    },
    "meta": {
        "version": "2.0"
    }
}
```

### HTML Responses (Web)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Sample Study</title>
</head>
<body>
    <div class="study">
        <h1>Sample Study</h1>
        <p>Created: January 1, 2023</p>
    </div>
</body>
</html>
```

## Error Handling Differences

### Web Error Responses

Web errors are displayed as HTML error pages:
```html
<!DOCTYPE html>
<html>
<head>
    <title>404 Not Found</title>
</head>
<body>
    <h1>404 Not Found</h1>
    <p>The requested page could not be found.</p>
</body>
</html>
```

### API Error Responses

API errors are returned as JSON objects:
```json
{
    "error": true,
    "message": "The requested resource could not be found.",
    "code": 404
}
```

---

**See Also:**
- [API_NEGOTIATION_GUIDE.md](API_NEGOTIATION_GUIDE.md) - Implementation guide, testing, security, and best practices

**Document Version:** 1.1
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete