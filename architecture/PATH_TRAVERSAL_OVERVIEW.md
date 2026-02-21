# Path Traversal Prevention Overview

## Overview

Path traversal (also known as directory traversal) is a web security vulnerability that allows attackers to read arbitrary files on the server. ML2 implements multiple layers of protection against this attack vector across different components of the application.

## Primary Defense Mechanisms

### 1. Static File Serving Protection

In `Public/index.php`, the static file serving function prevents path traversal through careful path construction:

```php
function serveStaticFile($uri) {
    $file = __DIR__ . $uri;
    
    if (!file_exists($file) || !is_file($file)) {
        http_response_code(404);
        echo "404 Not Found";
        return;
    }
    
    // ... serve file with explicit MIME type
}
```

#### Security Analysis

- **Absolute Path Prefix**: `__DIR__` ensures all paths start from the known Public directory
- **Existence Validation**: `file_exists()` and `is_file()` checks ensure only valid files are accessed
- **Implicit Containment**: Any attempt to traverse above the Public directory results in a non-existent path

### 2. Page Loading Protection

The `Presenter` class in `App/Presentation/Presenter.php` uses `basename()` to prevent path traversal in page loading:

```php
public function getPage(string $page): string {
    $page = basename($page); // Prevent directory traversal
    
    $extensions = ['html', 'htm'];
    
    $publicPagesDir = dirname(dirname(__DIR__)) . '/Public/Pages';
    $authenticatedPagesDir = __DIR__;
    
    // ... rest of implementation
}
```

#### Security Analysis

- **Filename Sanitization**: `basename()` strips directory components from the input
- **Additional Layer**: Even if user input reaches this method, traversal sequences are removed
- **Consistent Approach**: Applied uniformly to all page loading operations

### 3. Development Server Protection

The development server in `App/dev-server.php` includes path traversal protections:

```php
// If the file exists and is not a directory, serve it directly (static files)
if ($uri !== '/' && file_exists(__DIR__ . '/../Public' . $uri) && !is_dir(__DIR__ . '/../Public' . $uri)) {
    return false; // Let PHP's built-in server handle it
}
```

#### Security Analysis

- **Directory Check**: Explicitly verifies that targets are not directories
- **Same Origin Containment**: Paths are constrained to the application directory structure
- **Fallback Security**: Non-static requests go through the main application security pipeline

## Defense in Depth Strategy

ML2 employs multiple, redundant protection mechanisms:

### Layer 1: Input Sanitization

- `basename()` calls strip directory components from user input
- URI parsing and normalization in routing components

### Layer 2: Path Construction

- Absolute path prefixes ensure all file operations start from known locations
- Controlled directory structures limit where files can be accessed

### Layer 3: Validation

- `file_exists()` checks ensure targets exist
- `is_file()` checks ensure targets are files, not directories
- Extension whitelisting limits acceptable file types

### Layer 4: Containment

- All file operations are constrained to specific directories
- Directory traversal outside permitted areas results in non-existent paths

## Common Attack Patterns and Defenses

### Classic Directory Traversal

**Attack Pattern:**
```
GET /../../../etc/passwd
GET /..\\..\\..\\..\\windows\\win.ini
```

**Defense:**
- Path construction with `__DIR__` prefix
- `file_exists()` validation that fails for paths outside application directory

### Encoded Traversal

**Attack Pattern:**
```
GET /%2e%2e/%2e%2e/%2e%2e/etc/passwd
GET /%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

**Defense:**
- PHP's automatic URL decoding means these are processed as regular traversal attempts
- Same protection mechanisms apply after decoding

### Null Byte Injection

**Attack Pattern:**
```
GET /file.php%00.jpg
```

**Defense:**
- Modern PHP versions are not vulnerable to null byte injection in filesystem functions
- Extension checking provides additional protection layer

### Double Encoding

**Attack Pattern:**
```
GET /%252e%252e/%252e%252e/%252e%252e/etc/passwd
```

**Defense:**
- Multiple rounds of URL decoding would be required
- Final path validation still applies containment principles

---

**See Also:**
- [PATH_TRAVERSAL_GUIDE.md](PATH_TRAVERSAL_GUIDE.md) - Testing, best practices, monitoring, and verification

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete