# API Content Negotiation Guide

## Controller Adaptation

Controllers can adapt their behavior based on the request type:

```php
class StudiesController {
    public function view(array $request) {
        $studyId = $request['params']['id'];
        $studyData = $this->app->study->getStudy($studyId);
        
        // Prepare data for both formats
        $responseData = [
            'study' => $studyData,
            'status' => 'success'
        ];
        
        // Add additional data for web responses
        if (!($request['is_api'] ?? false)) {
            $responseData['template'] = 'studies/view';
            $responseData['page_title'] = $studyData['title'];
        }
        
        return $responseData;
    }
}
```

## Middleware Considerations

Some middleware may behave differently for API vs web requests:

### Authentication Middleware

The `IdentityMiddleware` handles authentication for both web and API requests:

```php
class IdentityMiddleware implements MiddlewareInterface {
    public function handle(array $request): array {
        // Try session authentication first
        $identityData = $this->sessionRepository->getAuthData();

        // If no session, try Authorization header (API requests)
        if (empty($identityData) && isset($request['headers']['Authorization'])) {
            $identityData = $this->authenticateViaHeader($request['headers']['Authorization']);
        }

        // Set identity in request
        if (!empty($identityData)) {
            $request['identity'] = Identity::fromArray($identityData);
        } else {
            $request['identity'] = null;
        }

        return $request;
    }

    private function authenticateViaHeader(string $authHeader): ?array {
        // Stub: Future implementation for bearer tokens/API keys
        // Currently returns null - all API auth uses sessions
        return null;
    }
}
```

**Note:** Token-based API authentication is planned but not yet implemented. All API requests currently use session-based authentication.

### CSRF Protection Middleware

```php
class CsrfProtection implements MiddlewareInterface {
    public function handle(array $request): array {
        $method = $request['method'] ?? 'GET';

        // Detect API/JSON requests via multiple signals
        $acceptHeader = $request['headers']['Accept'] ?? '';
        $isApiOrJson = ($request['is_api'] ?? false)
            || strpos($acceptHeader, 'application/json') !== false
            || $this->hasAuthorizationHeader($request);

        // Only include CSRF token for non-API HTML responses
        $request['includeCsrfToken'] = !$isApiOrJson;

        if (!$isApiOrJson) {
            $request['csrfToken'] = $this->getToken();
        }

        // Skip CSRF validation for:
        // 1. Safe methods (GET, HEAD, OPTIONS)
        // 2. API requests WITH Authorization header
        if (!$this->requiresCsrfValidation($method, $request)) {
            return $request;
        }

        // Validate CSRF token for state-changing requests
        $token = $this->extractToken($request);
        if (!$this->validateToken($token, $request)) {
            throw new \RuntimeException('Invalid CSRF token', 403);
        }

        return $request;
    }
}
```

## Testing Considerations

### API Endpoint Testing

API endpoints should be tested with:
- Correct `Content-Type` headers (`application/json`)
- Proper authentication tokens
- JSON request bodies
- Expectation of JSON responses

### Web Endpoint Testing

Web endpoints should be tested with:
- Standard form data submission
- Session cookies
- Expectation of HTML responses
- Browser-compatible redirects

## Security Considerations

### Rate Limiting

API endpoints may require rate limiting:
- **Throttling**: Limit requests per time period
- **Quotas**: Daily/monthly request limits
- **Authentication**: Rate limits vary by authentication level

### CORS Handling

API endpoints may need CORS headers:
```php
if ($request['is_api'] ?? false) {
    header('Access-Control-Allow-Origin: *');
    header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE');
    header('Access-Control-Allow-Headers: Authorization, Content-Type');
}
```

## Performance Implications

### Response Size

- **JSON Responses**: Typically smaller, faster to transmit
- **HTML Responses**: Larger due to markup and styling, but can be cached more effectively

### Processing Overhead

- **JSON Encoding**: Minimal overhead for API responses
- **Template Rendering**: Higher overhead for web responses but enables rich UI

## Client-Side Considerations

### JavaScript API Clients

Frontend JavaScript can consume API endpoints:
```javascript
fetch('/api/studies/123', {
    headers: {
        'Authorization': 'Bearer ' + authToken,
        'Content-Type': 'application/json'
    }
})
.then(response => response.json())
.then(data => console.log(data));
```

### Mobile Applications

Mobile apps benefit from the JSON API format:
- **Parsing Efficiency**: Native JSON parsing
- **Bandwidth Usage**: Smaller payloads
- **Offline Storage**: Easier data caching

## Versioning Strategies

Future API versions could be implemented through:
- **URI Versioning**: `/api/v2/studies/123`
- **Header Versioning**: `Accept: application/vnd.ml2.v2+json`
- **Parameter Versioning**: `/api/studies/123?version=2`

## Monitoring and Analytics

### Request Tracking

Track API vs web requests separately for:
- **Usage Analytics**: Understand API adoption
- **Performance Monitoring**: Different performance profiles
- **Error Tracking**: Separate error reporting channels

### Quick Reference: API vs Web Requests

| Aspect | Web | API |
|--------|-----|-----|
| URL Prefix | None | `/api/` |
| Request Body | Form data → `$request['params']` | JSON → `$request['params']` |
| Auth Method | Session | Session (token auth stub) |
| Response Format | HTML | JSON |
| CSRF Protection | Required | Required (unless Auth header) |
## Best Practices

### Do
- Use the same controller logic for both API and web when possible
- Check `$request['is_api']` only when necessary for format differences
- Return consistent data structures
- Use proper HTTP status codes

### Don't
- Duplicate controller logic for API vs web
- Hardcode response formats
- Mix authentication methods inappropriately
- Skip validation for API requests

## Quick Reference

| Aspect | Web | API |
|--------|-----|-----|
| URL Prefix | None | `/api/` |
| Request Body | Form data | JSON |
| Auth Method | Session | Token |
| Response Format | HTML | JSON |
| CSRF Protection | Required | Not needed |
| Content-Type | `text/html` | `application/json` |

---

**See Also:**
- [API_NEGOTIATION_OVERVIEW.md](API_NEGOTIATION_OVERVIEW.md) - Overview, detection, and format differences

**Document Version:** 1.1
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
