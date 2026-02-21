# Tier 07: CsrfProtection

**Purpose**: Prevent Cross-Site Request Forgery (CSRF) attacks by validating tokens on all state-changing requests.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['csrf_valid']` | `bool` | Set to `true` when token validation succeeds |
| `$request['csrfToken']` | `string` | Generated token for forms (non-API requests only) |
| `$request['includeCsrfToken']` | `bool` | Flag indicating token should be included in responses |

## Responsibilities

1. **Token Generation**: Creates cryptographically secure CSRF tokens stored in session
2. **Token Validation**: Validates tokens on state-changing requests (POST, PUT, DELETE, PATCH)
3. **Multi-Source Extraction**: Reads tokens from POST data or HTTP headers
4. **API Exemption**: Skips validation for requests with Authorization headers
5. **Safe Method Bypass**: Skips validation for GET, HEAD, OPTIONS requests

## CSRF Methods

State-changing HTTP methods that require token validation:

```php
private const CSRF_METHODS = ['POST', 'PUT', 'DELETE', 'PATCH'];
```

## Token Sources

The middleware checks for tokens in this order:

1. **POST Data**: `$request['post']['csrf_token']`
2. **HTTP Headers** (for AJAX):
   - `X-CSRF-Token`
   - `X-XSRF-TOKEN`
   - `X-CSRF-TOKEN`

```php
private function extractToken(array $request): ?string {
    // Primary: POST data
    $post = $request['post'] ?? [];
    if (isset($post[self::CSRF_FIELD])) {
        return trim($post[self::CSRF_FIELD]);
    }
    
    // Secondary: Headers
    $headers = $request['headers'] ?? [];
    foreach (self::CSRF_HEADERS as $headerName) {
        if (isset($headers[$headerName])) {
            return trim($headers[$headerName]);
        }
    }
    
    return null;
}
```

## API Request Exemption

Requests with Authorization headers are exempt from CSRF validation (they use token-based auth):

```php
private function requiresCsrfValidation(string $method, array $request): bool {
    // Skip safe methods
    if (!in_array(strtoupper($method), self::CSRF_METHODS, true)) {
        return false;
    }
    
    // Skip API requests with Authorization header
    if ($this->hasAuthorizationHeader($request)) {
        return false;
    }
    
    return true;
}
```

## Token Generation

Tokens are cryptographically secure 64-character hex strings:

```php
public function generateToken(): string {
    $this->ensureSession();

    try {
        $token = bin2hex(random_bytes(32));  // 64 hex chars
    } catch (\Exception $e) {
        // Fallback to openssl if random_bytes fails
        $bytes = openssl_random_pseudo_bytes(32);
        $token = bin2hex($bytes);
    }

    $this->sessionRepository->set('csrf_token', $token);
    return $token;
}
```

## Token Validation

Uses `hash_equals()` for timing-attack-safe comparison:

```php
private function validateTokenInternal(string $token): bool {
    $this->ensureSession();

    if ($token === '') {
        return false;
    }

    $sessionToken = $this->sessionRepository->get('csrf_token');
    if (!is_string($sessionToken) || $sessionToken === '') {
        return false;
    }

    return hash_equals($sessionToken, $token);
}
```

## Flow Diagram

```
Request arrives
       │
       ▼
┌─────────────────────────────┐
│ Is API request?             │─Yes──► Skip validation, attach token
│ (Authorization header)      │        for forms if needed
└─────────────────────────────┘
       │ No
       ▼
┌─────────────────────────────┐
│ Is safe method?             │─Yes──► Skip validation, attach token
│ (GET/HEAD/OPTIONS)          │        for forms
└─────────────────────────────┘
       │ No (POST/PUT/DELETE/PATCH)
       ▼
┌─────────────────────────────┐
│ Extract token from POST     │
│ or headers                  │
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ Token present?              │─No──► Log failure, throw 403
└─────────────────────────────┘
       │ Yes
       ▼
┌─────────────────────────────┐
│ Compare with session using  │
│ hash_equals()               │
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ Match?                      │─No──► Log failure, throw 403
└─────────────────────────────┘
       │ Yes
       ▼
┌─────────────────────────────┐
│ Set $request['csrf_valid']  │
│ = true                      │
└─────────────────────────────┘
       │
       ▼
   Continue
```

## Form Usage

### Standard HTML Form

```html
<form action="/studies/save" method="POST">
    <input type="hidden" name="csrf_token" value="<?= $csrfToken ?>">
    <input type="text" name="title" placeholder="Study Title">
    <button type="submit">Save</button>
</form>
```

### With Method Override

```html
<form action="/studies/42" method="POST">
    <input type="hidden" name="csrf_token" value="<?= $csrfToken ?>">
    <input type="hidden" name="_method" value="DELETE">
    <button type="submit">Delete Study</button>
</form>
```

## AJAX/Fetch Usage

### With Header

```javascript
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

fetch('/studies/42', {
    method: 'PUT',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': csrfToken,
    },
    body: JSON.stringify({ title: 'Updated Title' }),
});
```

### With POST Data

```javascript
const formData = new FormData();
formData.append('csrf_token', csrfToken);
formData.append('title', 'Updated Title');

fetch('/studies/42', {
    method: 'POST',
    body: formData,
});
```

## Controller Integration

Controllers receive the token via `$request` for rendering in templates:

```php
public function create($app, array $request): array {
    return [
        'csrfToken' => $request['csrfToken'] ?? null,
        // ... other data
    ];
}
```

Template usage:

```php
<form method="POST">
    <input type="hidden" name="csrf_token" value="<?= $csrfToken ?>">
    <!-- form fields -->
</form>
```

## Error Handling

| Exception | Code | Condition |
|-----------|------|-----------|
| `RuntimeException` | 403 | Missing CSRF token on state-changing request |
| `RuntimeException` | 403 | Token mismatch (validation failure) |

## Security Logging

Failed validation attempts are logged with client information:

```php
private function logCsrfFailure(array $request): void {
    $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    $method = $request['method'] ?? 'unknown';
    $uri = $request['uri'] ?? 'unknown';
    
    error_log("CSRF validation failed - IP: {$ip}, Method: {$method}, URI: {$uri}");
}
```

## Token Lifecycle

- **Generation**: New token created on first access if none exists
- **Storage**: Stored in PHP session under key `csrf_token`
- **Validation**: Compared against submitted token using timing-safe comparison
- **Rotation**: Token persists for session lifetime; regenerated when explicitly requested

## Session Management

The middleware ensures an active session exists before token operations:

```php
private function ensureSession(): void {
    if (session_status() !== PHP_SESSION_ACTIVE) {
        session_start();
    }
}
```

## Dependencies

- **SessionRepository**: `/App/Data/Repositories/SessionRepository.php`

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 06: UserContext](06_context.md)
- [Tier 04: HttpMethodOverride](04_http_override.md)
