# API Development

ML2 provides both internal APIs for the application and external API clients for third-party services. This guide covers both creating API endpoints and consuming external APIs.

## Overview

API responses are determined by content negotiation:
- **HTML**: Default for browser requests
- **JSON**: Returned when `Accept: application/json` header is present or `$request['is_api']` is true

## Content Negotiation

The `handleContentNegotiation()` method in `App.php` determines response format:

```php
$acceptHeader = $request['headers']['Accept'] ?? '';
$acceptsJson = strpos($acceptHeader, 'application/json') !== false;
$isApi = isset($request['is_api']) && $request['is_api'] === true;
$wantJson = $acceptsJson || $isApi;
```

### HTML Response

```php
return ResponseBuilder::success(
    data: ['studies' => $studies],
    template: 'Studies/list'  // Template triggers HTML rendering
);
```

### JSON Response

```php
return ResponseBuilder::success(
    data: ['studies' => $studies]
    // No template = JSON response when is_api=true
);
```

## Creating API Endpoints

### Controller Method

```php
public function list(array $request): array {
    $userId = (int)($request['identity']->id ?? 0);
    $studies = $this->studyService->getByUser($userId);
    
    // Works for both HTML and JSON
    return ResponseBuilder::success(
        data: ['studies' => $studies],
        template: 'Studies/list'  // Used for HTML, ignored for JSON
    );
}
```

### Route Configuration

Add to `Data/routes.json`:

```json
{
    "/api/studies": {
        "controller": "StudiesController",
        "method": "list",
        "access": "authenticated"
    }
}
```

### Testing API Endpoint

```bash
# JSON request
curl -H "Accept: application/json" \
     -H "Cookie: PHPSESSID=your_session" \
     http://localhost/api/studies

# Or with is_api flag
curl -H "Cookie: PHPSESSID=your_session" \
     "http://localhost/api/studies?is_api=1"
```

## External API Clients

API clients live in `App/Data/Clients/` and wrap HTTP calls to external services.

### Client Structure

```php
<?php
declare(strict_types=1);

namespace MemorizeLive\App\Data\Clients;

use MemorizeLive\App\Data\HttpClient;

class MyApiClient {
    private HttpClient $httpClient;
    
    public function __construct(
        private array $config,
        private ?HttpClient $httpClient = null
    ) {
        if ($this->httpClient === null) {
            $baseUrl = $this->config['apis']['myapi'] ?? 'https://api.example.com';
            $timeout = $this->config['apis']['timeout'] ?? 30;
            $this->httpClient = new HttpClient($baseUrl, (int)$timeout);
        }
    }
    
    public function getData(string $id): ?array {
        try {
            $response = $this->httpClient->get("/api/v1/data/{$id}");
            return $this->processResponse($response);
        } catch (\Exception $e) {
            error_log("API error: " . $e->getMessage());
            return null;
        }
    }
    
    private function processResponse(array $response): ?array {
        if (!isset($response['result'])) {
            return null;
        }
        return $response['result'];
    }
}
```

### HttpClient

Base HTTP client for making requests:

```php
use MemorizeLive\App\Data\HttpClient;

$client = new HttpClient('https://api.example.com', 30);

// GET request
$response = $client->get('/endpoint', ['param' => 'value']);

// POST request
$response = $client->post('/endpoint', ['data' => 'value']);
```

## Existing API Clients

### VerseApiClient

Bible verse operations:

```php
$client = new VerseApiClient($config);

// Normalize reference
$normalized = $client->normalize('John 3:16');
// Returns: ['book' => 'John', 'chapter' => 3, 'verse' => 16]

// Convert to numeric
$numeric = $client->numeric('John 3:16');
// Returns: ['result' => [['start' => '043003016', ...]]]

// Validate reference
$isValid = $client->validate('John 3:16'); // true/false

// Get verse text
$text = $client->getVerseText('043003016');
```

### ScheduleApiClient

Schedule generation:

```php
$client = new ScheduleApiClient($config);

// Generate schedule
$schedule = $client->generate([
    'type' => 'pattern',
    'pattern' => [1, 3, 7, 14, 30],
    'timezone' => 'America/New_York'
]);
```

## API Response Format

### Success Response

```json
{
    "success": true,
    "data": {
        "studies": [...],
        "pagination": {...}
    }
}
```

### Error Response

```json
{
    "error": true,
    "status": 422,
    "message": "Validation failed",
    "data": {
        "errors": {
            "title": "Title is required"
        }
    }
}
```

## Best Practices

### 1. Always Handle Errors

```php
public function fetchData(): ?array {
    try {
        return $this->httpClient->get('/data');
    } catch (\Exception $e) {
        // Log error
        error_log("API Error: " . $e->getMessage());
        // Return null or safe default
        return null;
    }
}
```

### 2. Use Config for URLs

```php
// Good - Configurable
$baseUrl = $this->config['apis']['myapi'] ?? 'https://default.example.com';

// Bad - Hardcoded
$baseUrl = 'https://api.example.com';
```

### 3. Validate Responses

```php
private function processResponse(array $response): ?array {
    if (!is_array($response) || !isset($response['result'])) {
        return null;
    }
    return $response['result'];
}
```

### 4. Set Appropriate Timeouts

```php
// Fast endpoint
$client = new HttpClient($url, 5);

// Slow endpoint
$client = new HttpClient($url, 30);
```

### 5. Use Type Declarations

```php
public function getVerse(string $reference): ?array {
    // Returns array or null
}
```

## Testing API Clients

### Mock HttpClient

```php
class MockHttpClient extends HttpClient {
    private array $responses = [];
    
    public function setResponse(string $url, array $response): void {
        $this->responses[$url] = $response;
    }
    
    public function get(string $endpoint, array $params = []): array {
        $url = $endpoint . '?' . http_build_query($params);
        return $this->responses[$url] ?? ['error' => 'Not found'];
    }
}

// Usage
$mock = new MockHttpClient();
$mock->setResponse('/api/v1/data/123', ['result' => ['id' => 123]]);

$client = new MyApiClient($config, $mock);
$result = $client->getData('123');
assert($result['id'] === 123);
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
