# ResponseBuilder API

The `ResponseBuilder` provides a standardized way to construct HTTP responses across the application. It supports both JSON and HTML responses through the use of an optional `$template` parameter.

## Method Signatures

### `success()`
General success response (200 OK).
```php
public static function success(
    mixed $data = null,
    string $message = 'Success',
    ?string $template = null,
    ?string $redirectUrl = null,
    int $status = 200
): array
```

### `created()`
Resource created response (201 Created).
```php
public static function created(
    mixed $data = null,
    string $message = 'Created',
    ?string $template = null,
    ?string $redirectUrl = null
): array
```

### `error()`
General error response.
```php
public static function error(
    string $message,
    int $status = 400,
    mixed $data = null,
    ?string $template = null
): array
```

### `unauthorized()`
Authentication required (401 Unauthorized).
```php
public static function unauthorized(
    string $message = 'Unauthorized',
    mixed $data = null,
    ?string $template = null
): array
```

### `forbidden()`
Access denied (403 Forbidden).
```php
public static function forbidden(
    string $message = 'Forbidden',
    mixed $data = null,
    ?string $template = null
): array
```

### `notFound()`
Resource not found (404 Not Found).
```php
public static function notFound(
    string $message = 'Not Found',
    mixed $data = null,
    ?string $template = null
): array
```

### `methodNotAllowed()`
HTTP method not allowed (405 Method Not Allowed).
```php
public static function methodNotAllowed(
    string $message = 'Method not allowed',
    mixed $data = null,
    ?string $template = null
): array
```

## The `$template` Parameter
The `$template` parameter allows for custom error or success views while maintaining a standard response structure. 

- If `$template` is provided, the rendering engine will attempt to render that specific view.
- This allows controllers to return specific error pages (e.g., a custom 404 page) or success pages instead of generic responses.
- Even when a template is used, the returned array retains its structured format (`status`, `message`, `data`, etc.), allowing the middleware or presenter to handle it correctly based on the request type (e.g., HTMX vs. standard GET).
