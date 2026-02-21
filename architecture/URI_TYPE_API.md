# URI Type: API Routes

## Classification Logic

API routes are a special case that are initially detected as heavy routes but receive special handling for content negotiation.

API routes are detected by the `/api/` prefix:

```php
// Router.php::match() - API detection
if (strpos($uri, '/api/') === 0) {
    $isApi = true;
    $uri = str_replace('/api/', '/', $uri); // Strip /api prefix
    $request['is_api'] = true; // Flag for controller
}
```

## Content Negotiation

API responses are formatted as JSON rather than HTML:

```php
// App.php::handleContentNegotiation() - Response formatting
private function handleContentNegotiation($output, array $request): string {
    if ($request['is_api']) {
        return json_encode($output, JSON_PRETTY_PRINT);
    }
    return $this->presenter->render($output['template'], $output['data']);
}
```

## Request/Response Format Differences

| Aspect | Web Routes | API Routes |
|--------|------------|------------|
| Path Prefix | None | `/api/` |
| Request Format | Form data/Query params | JSON body/Query params |
| Response Format | HTML | JSON |
| Authentication | Session-based | Token-based |
| Error Format | HTML error pages | JSON error objects |

## URI Transformation

```
Original: GET /api/studies/123
Processed: GET /studies/123 (with is_api=true flag)
```

The `/api/` prefix is stripped to normalize the URI for route matching, allowing the same controller logic to handle both API and web requests.

---

**See Also:**
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Overview of all URI types
- [API_NEGOTIATION_OVERVIEW.md](API_NEGOTIATION_OVERVIEW.md) - Detailed API content negotiation

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete