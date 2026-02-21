# Partials

Partials are PHP classes that render specific page sections — header, menu, notifications, footer. They solve a single problem: common page sections defined once and reused across all templates.

## Architecture

```
App/Presentation/Partials/
├── HeaderPartial.php
├── MenuPartial.php
├── NotificationsPartial.php
└── FooterPartial.php
```

Each partial is a static class with a `render()` method that includes a template file and returns the output.

## How Partials Work

A layout template calls each partial in sequence:

```php
<?= HeaderPartial::render($title) ?>
<?= MenuPartial::render($menu, $currentUri) ?>
<?= NotificationsPartial::render() ?>

<main class="container">
    <?= $this->section('content') ?>
</main>

<?= FooterPartial::render() ?>
```

The partial class loads its template file, captures the output, and returns it:

```php
class HeaderPartial {
    public static function render(string $title = 'Application'): string {
        ob_start();
        include __DIR__ . '/templates/header.php';
        return ob_get_clean();
    }
}
```

## Creating a Custom Partial

1. Create the partial class in `App/Presentation/Partials/`:

```php
namespace Application\App\Presentation\Partials;

class SidebarPartial {
    public static function render(array $widgets): string {
        ob_start();
        include __DIR__ . '/templates/sidebar.php';
        return ob_get_clean();
    }
}
```

2. Create the template file:

```php
<!-- templates/sidebar.php -->
<aside class="sidebar">
    <?php foreach ($widgets as $widget): ?>
        <div class="widget">
            <h3><?= htmlspecialchars($widget['title']) ?></h3>
            <p><?= htmlspecialchars($widget['content']) ?></p>
        </div>
    <?php endforeach; ?>
</aside>
```

3. Call it from your layout:

```php
<?= SidebarPartial::render($this->data['widgets'] ?? []) ?>
```

## Best Practices

- **Sanitize inputs**: Use `htmlspecialchars()` on any user-supplied data passed into partials
- **Keep templates simple**: Templates display data, they don't process it
- **Use output buffering**: Always capture with `ob_start()` / `ob_get_clean()`
- **Pass explicit variables**: Don't reach for globals inside partials

---

**Related Docs:**
- [GUIDE_CREATING_VIEWS_AND_TEMPLATES.md](GUIDE_CREATING_VIEWS_AND_TEMPLATES.md)
- [DLPR Pattern](../architecture/DLPR_PATTERN.md)
