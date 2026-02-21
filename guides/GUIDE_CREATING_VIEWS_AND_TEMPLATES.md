# Guide: Creating Views and Templates

Template naming, Plates engine usage, fallback behavior, and template organization.

## Template Engine

Uses **League/Plates** PHP templating library. Falls back to basic PHP if unavailable.

## Template Locations

```
App/Presentation/
├── layout.php                 # Main layout (used by $this->layout())
├── Studies/                   # Feature directory
│   ├── list.php               # Generic fallback
│   ├── form.php               # Unified create/edit
│   ├── focused/               # Interface-specific
│   │   └── list.php
│   ├── normal/
│   │   └── list.php
│   ├── power/
│   │   └── list.php
│   └── admin/
│       └── list.php
├── Schedules/                 # Feature directory with interface resolution
├── Notifications/             # Feature directory with interface resolution
├── User/
│   ├── edit.php
│   └── view.php
└── Partials/                  # Reusable components
    ├── HeaderPartial.php
    ├── MenuPartial.php
    ├── FooterPartial.php
    └── theme/
        ├── application/
        │   ├── header.php
        │   ├── menu.php
        │   └── footer.php
        └── original/
            ├── header.php
            ├── menu.php
            └── footer.php
```

## Template Naming Conventions

| Pattern | Example | Purpose |
|---------|---------|---------|
| `{Feature}/{action}.php` | Studies/list.php | Standard CRUD views |
| `{Feature}/{interface}/{action}.php` | Studies/focused/list.php | Interface-specific |
| `{Feature}/form.php` | Studies/form.php | Unified create/edit |
| `{Feature}/success.php` | Studies/success.php | After-action page |

## Plates Engine Basics

### Layout Inheritance
```php
<?php $this->layout('layout', ['title' => 'Page Title']) ?>

<h1>Content goes here</h1>
```

### Output Escaping
```php
<?= $this->e($variable) ?>     <!-- Escaped output -->
<?php echo $variable; ?>       <!-- Raw output (avoid) -->
```

### Foreach Loop
```php
<?php foreach ($items as $item): ?>
    <div><?= $this->e($item['name']) ?></div>
<?php endforeach; ?>
```

### Sections (for layout)
```php
<?php $this->start('content') ?>
    <!-- This goes into layout's $this->section('content') -->
<?php $this->stop() ?>
```

## Template Resolution

Presenter.php resolves templates with interface fallback for specific directories (`Studies`, `Schedules`, `Notifications`):

```
Template: 'Studies/list'
    ├─ Get interface level from session
    └─ Try: Studies/{level}/list.php → fallback: Studies/list.php

Template: 'Studies/create' (no interface variants)
    └─ Use: Studies/create.php
```

## Fallback Behavior

1. **Interface fallback:** Interface-specific → Generic (Applied to `Studies`, `Schedules`, `Notifications`)
2. **Direct match:** Match directly in `App/Presentation/`
3. **Missing template:** Throws exception

### Fallback Example
```
Request: Studies/list (interface=1 focused)
    ↓
Try: App/Presentation/Studies/focused/list.php
    ↓ (not found)
Try: App/Presentation/Studies/list.php
    ↓ (found)
Returns: Studies/list.php
```

## Creating a New Template

### 1. Create Template File
```php
<!-- App/Presentation/MyFeature/view.php -->
<?php $this->layout('layout', ['title' => $title]) ?>

<section class="section">
    <div class="container">
        <h1 class="title"><?= $this->e($title) ?></h1>
        <div class="content">
            <?= $this->e($description) ?>
        </div>
    </div>
</section>
```

### 2. Add Render Method to Presenter
```php
// App/Presentation/Presenter.php
public function renderMyFeatureView(array $data): string {
    return $this->render('MyFeature/view', $data);
}
```

### 3. Create Controller Method
```php
// App/Routes/Controllers/MyFeatureController.php
public function view($app, $request) {
    $id = $request['params']['id'] ?? 0;
    $feature = $app->myService->getById($id);
    
    return [
        'template' => 'MyFeature/view',
        'data' => [
            'title' => $feature['title'],
            'description' => $feature['description']
        ]
    ];
}
```

### 4. Add Route
```json
// Data/routes.json
{
    "/myfeature/view": {
        "controller": "MyFeatureController",
        "method": "view"
    }
}
```

## Using Partials

### Header/Menu/Footer
```php
<?php $this->layout('layout') ?>
```

The layout includes header, menu, and footer automatically via partials.

### Custom Partial
```php
<!-- In your template -->
<?php echo MyNamespace\SomePartial::render($data); ?>
```

## Theme Integration

Themes affect header, menu, and footer partials:

```php
// layout.php
$theme = $this->data['theme'] ?? 'default';
echo HeaderPartial::render($theme, $title);
```

Theme files located at `Partials/theme/{theme}/header.php|menu.php|notifications.php|footer.php`.

## Public Pages (3-Stage Fallback)

Public pages under `/page/*` use 3-stage fallback:

### Stage 1: HTML File (simplest)
```
Public/Pages/about.html → served at /page/about
```

### Stage 2: PHP File (dynamic)
```
Public/Pages/pricing.php → served at /page/pricing
```

### Stage 3: Plates Template (full control)
```
App/Presentation/Pages/about.plates.php
```

## Best Practices

1. **Always escape user data:** Use `$this->e()`
2. **Use layout inheritance:** Don't repeat header/footer
3. **Keep logic minimal:** Templates display, don't process
4. **Interface variants:** Only create when needed, fallback works
5. **Naming:** Lowercase with underscores or camelCase

## Testing Templates

```bash
# Start server
./serve.sh

# Test public page
curl http://localhost:8080/page/about

# Test authenticated page
curl http://localhost:8080/studies

# Test with different interface
# (Set interface level in session, then test)

**Status:** Accurate and Complete
**Last Updated:** 2025-02-08
```
