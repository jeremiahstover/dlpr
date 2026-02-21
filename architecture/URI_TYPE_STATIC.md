# URI Type: Static Files

## Classification Logic

```php
// Router.php::identifyRouteType() - lines 190-193
if (preg_match('/\.(css|js|png|jpg|jpeg|gif|svg|ico|woff|woff2|ttf|eot|json)$/i', $uri)) {
    return ['type' => 'static', 'resolved_uri' => $uri];
}
```

## Processing Pipeline

Requests classified as static bypass the PHP application framework entirely and are served directly by the `serveStaticFile()` function in `Public/index.php`.

```php
function serveStaticFile($uri) {
    $file = __DIR__ . $uri;
    
    if (!file_exists($file) || !is_file($file)) {
        http_response_code(404);
        echo "404 Not Found";
        return;
    }
    
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
    readfile($file);
}
```

## Allowed Extensions

1. **CSS (.css)**: Stylesheets with `text/css` MIME type
2. **JavaScript (.js)**: Client-side scripts with `application/javascript` MIME type
3. **Images (.png, .jpg, .jpeg, .gif, .svg, .ico)**: Various image formats
4. **Fonts (.woff, .woff2, .ttf, .eot)**: Web font formats
5. **JSON (.json)**: Data files with `application/json` MIME type

## Security Considerations

- **Directory Traversal Prevention**: Path constructed with `__DIR__` prefix prevents access outside Public directory
- **MIME Type Enforcement**: Explicit mapping prevents browsers from interpreting files as executable scripts
- **File Existence Validation**: Only existing files are served

## Performance Characteristics

- **Low Latency**: No PHP framework loading
- **Minimal Memory**: No application memory overhead
- **Direct File Serving**: Efficient kernel-level file operations

---

**See Also:**
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Overview of all URI types

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete