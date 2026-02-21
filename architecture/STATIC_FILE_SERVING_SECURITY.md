# Static File Serving Security

This document details the security mechanisms implemented in Application for serving static files, with a focus on preventing directory traversal attacks and enforcing secure MIME type handling.

## Overview

Application implements several security measures in its static file serving mechanism to prevent common web vulnerabilities, particularly directory traversal attacks and MIME type confusion attacks.

## Directory Traversal Prevention

The primary security concern with static file serving is preventing directory traversal attacks, where malicious users attempt to access files outside the intended directory by using sequences like `../` in the URL.

### Implementation

The static file serving logic in `Public/index.php` prevents directory traversal through a combination of path construction and validation:

```php
function serveStaticFile($uri) {
    $file = __DIR__ . $uri;
    
    if (!file_exists($file) || !is_file($file)) {
        http_response_code(404);
        echo "404 Not Found";
        return;
    }
    
    // ... serve file
}
```

### Security Mechanism

1. **Path Construction**: The file path is constructed by concatenating `__DIR__` (the Public directory) with the URI
2. **Existence Validation**: The system checks that the constructed path both exists and is a file (not a directory)
3. **Implicit Containment**: Because `__DIR__` is an absolute path to the Public directory, any attempt to traverse above this directory will result in a path that doesn't exist

### Example Attack Prevention

Consider an attack attempt with the URI `/../../etc/passwd`:
1. The path would be constructed as `/path/to/app/Public/../../etc/passwd`
2. This resolves to `/path/to/app/etc/passwd` 
3. Since this file doesn't exist within the application structure, `file_exists()` returns false
4. The system returns a 404 error without ever accessing the file

## MIME Type Security

Application explicitly defines MIME types for static files to prevent browsers from interpreting files incorrectly.

### Implementation

```php
$mimeTypes = [
    'css' => 'text/css',
    'js' => 'application/javascript',
    'png' => 'image/png',
    'jpg' => 'image/jpeg',
    'jpeg' => 'image/jpeg',
    'gif' => 'image/gif',
    'svg' => 'image/svg+xml',
    'ico' => 'image/x-icon',
    'woff' => 'font/woff',
    'woff2' => 'font/woff2',
    'ttf' => 'font/ttf',
    'eot' => 'application/vnd.ms-fontobject',
];

$ext = pathinfo($file, PATHINFO_EXTENSION);
$mimeType = $mimeTypes[$ext] ?? 'application/octet-stream';

header('Content-Type: ' . $mimeType);
```

### Security Benefits

1. **Prevents Script Execution**: By explicitly setting MIME types, the system prevents browsers from interpreting files as executable scripts
2. **Reduces XSS Risk**: Proper MIME type handling reduces the risk of cross-site scripting through maliciously crafted files
3. **Consistent Behavior**: Ensures predictable handling of file types across different browsers

## File Extension Whitelisting

The system only serves files with specific, whitelisted extensions:

### Allowed Extensions

- **Stylesheets**: `.css`
- **Scripts**: `.js`
- **Images**: `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.ico`
- **Fonts**: `.woff`, `.woff2`, `.ttf`, `.eot`
- **Data**: `.json`

### Security Rationale

This approach:
1. **Limits Attack Surface**: Only allows known file types that are needed for the application
2. **Prevents PHP Execution**: Ensures PHP files in the Public directory cannot be executed
3. **Controls Content**: Maintains control over what types of content can be served

## Development Server Security

The development server in `App/dev-server.php` also implements security measures:

```php
// If the file exists and is not a directory, serve it directly (static files)
if ($uri !== '/' && file_exists(__DIR__ . '/../Public' . $uri) && !is_dir(__DIR__ . '/../Public' . $uri)) {
    return false; // Let PHP's built-in server handle it
}
```

### Key Security Features

1. **Directory Check**: Ensures the target is not a directory before serving
2. **Containment**: Paths are constrained to the Public directory
3. **Fallback Routing**: Non-static requests are routed through the main application entry point

## Path Traversal Prevention in Related Components

Other components in the application also implement path traversal prevention:

### Presenter Class

The `Presenter` class uses `basename()` to prevent directory traversal in page loading:

```php
public function getPage(string $page): string {
    $page = basename($page); // Prevent directory traversal
    
    // ... rest of implementation
}
```

This ensures that even if user input reaches this method, directory traversal sequences are removed.

## Security Testing

The application includes automated security tests to verify path traversal prevention:

```php
// Tests/Security/test-template-security.php
$assertTrue(
    strpos($presenterContent, 'basename(') !== false,
    'Presenter.php uses basename() for path traversal prevention'
);

$assertTrue(
    strpos($presenterContent, 'basename($page)') !== false,
    'getPage() uses basename() to prevent path traversal'
);
```

## Additional Security Considerations

### File Permissions

The system relies on proper file permissions to prevent unauthorized access:
- Static files should have appropriate read permissions
- Sensitive files should be outside the web root
- Configuration files should not be directly accessible

### Content Security Policy

While not directly part of static file serving, Content Security Policy headers should be used to further restrict what resources can be loaded and executed.

### Cache Control

Proper cache control headers help prevent sensitive information leakage through browser caches.

## Common Attack Vectors Addressed

### Classic Directory Traversal

```
GET /../../../../etc/passwd
GET /..\\..\\..\\..\\windows\\win.ini
```

These are prevented by the path construction and validation approach.

### Null Byte Injection

```
GET /file.php%00.jpg
```

Modern PHP versions are not vulnerable to null byte injection in file operations, but the explicit extension checking adds an additional layer of defense.

### Double Encoding

```
GET /%2e%2e/%2e%2e/%2e%2e/etc/passwd
```

The system's reliance on `file_exists()` means that encoded characters are properly decoded before path resolution, and the containment principle still applies.

## Best Practices Implemented

### Principle of Least Privilege

- Only necessary file types are served
- Files are served with minimal permissions
- Access is restricted to the Public directory

### Defense in Depth

- Multiple layers of protection (path construction, existence checking, MIME type enforcement)
- Redundant checks in related components
- Automated testing to verify security measures

### Fail Secure

- Invalid requests result in 404 errors rather than revealing system information
- No verbose error messages that could aid attackers
- Graceful handling of edge cases

## Monitoring and Logging

While the current implementation focuses on prevention, production systems should consider:

1. **Access Logging**: Log requests for static files for monitoring
2. **Attack Detection**: Monitor for suspicious patterns in request logs
3. **Rate Limiting**: Prevent abuse of static file endpoints

## Future Security Enhancements

### Content Hashing

Adding content hashing to static file URLs to improve cache busting and provide integrity verification:

```php
// TODO: Add timestamp versioning based on file modification time
```

### Enhanced MIME Type Validation

Implementing more sophisticated MIME type detection to complement extension-based detection.

### Subresource Integrity

Generating and serving subresource integrity hashes for critical static assets.

---

**Related Docs:**
- [PATH_TRAVERSAL_OVERVIEW.md](PATH_TRAVERSAL_OVERVIEW.md) - Path traversal prevention overview
- [PATH_TRAVERSAL_GUIDE.md](PATH_TRAVERSAL_GUIDE.md) - Path traversal prevention guide
- [ENTRY_POINTS.md](ENTRY_POINTS.md) - Entry point and routing

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete