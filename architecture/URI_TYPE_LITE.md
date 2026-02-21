# URI Type: Lite Routes

## Classification Logic

```php
// Pages directory routes (lite) - handle /page/* without .html
if (preg_match('#^/page/[^.]+$#', $uri) || preg_match('#^/page/[^.]+$#', $resolvedUri)) {
    return ['type' => 'lite', 'resolved_uri' => $resolvedUri];
}
```

## Processing Pipeline

Lite routes are handled by `App/Lite.php` which implements a 3-stage fallback system:

```php
// Lite.php - 3-stage fallback logic
public function handle(string $resolvedUri): string {
    // Stage 1: Direct HTML serving
    $htmlPath = __DIR__ . '/Presentation/Templates/Pages/' . basename($resolvedUri) . '.html';
    if (file_exists($htmlPath)) {
        return file_get_contents($htmlPath);
    }
    
    // Stage 2: PHP template processing  
    $templatePath = __DIR__ . '/Presentation/Templates/Pages/' . basename($resolvedUri) . '.php';
    if (file_exists($templatePath)) {
        return $this->presenter->render($templatePath, $data);
    }
    
    // Stage 3: Plates template engine
    return $this->presenter->render('pages/' . basename($resolvedUri), $data);
}
```

## Three-Stage Fallback System

1. **Direct HTML Serving**: Serve pre-rendered HTML files directly from disk
2. **PHP Template Processing**: Execute PHP templates with basic data injection
3. **Plates Template Engine**: Use the full Plates templating engine for complex layouts

## Performance Comparison

| Stage | Latency | Memory | Use Case |
|-------|---------|--------|----------|
| HTML | Lowest | Minimal | Static content |
| PHP | Medium | Low | Simple dynamic content |
| Plates | Highest | Moderate | Complex layouts |

## When to Use Each Stage

- **HTML**: Static informational pages
- **PHP**: Pages with simple dynamic elements
- **Plates**: Pages requiring complex layouts or component reuse

---

**See Also:**
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Overview of all URI types

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete