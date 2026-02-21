# Asset Pipeline

Assets are served directly from `Public/Assets/` without build tools or bundling. No compilation step, no preprocessors — files are served as-is via the static URI type.

## Directory Structure

```
Public/Assets/
├── css/
│   ├── app.css          # Primary stylesheet
│   └── admin.css        # Role-specific styles (optional)
├── js/
│   ├── site.js          # Global JavaScript
│   └── myfeature.js     # Page-specific scripts
└── img/
    ├── favicon.ico
    └── ...
```

## CSS

### Loading Stylesheets

Load in the header partial:

```html
<link rel="stylesheet" href="/Assets/css/app.css">
```

### Role-Based CSS

Additional CSS can load conditionally based on user role:

```php
if ($userRole === 'admin') {
    echo '<link rel="stylesheet" href="/Assets/css/admin.css">';
}
```

### CSS Best Practices

Use CSS variables for design tokens:

```css
:root {
    --primary: #0055ff;
    --background: #ffffff;
    --text: #1a1a1a;
    --spacing: 1rem;
}
```

Follow BEM naming, mobile-first media queries, avoid `!important`.

## JavaScript

### Global Scripts

Add to `Public/Assets/js/site.js` for functionality needed on every page:

```javascript
document.addEventListener('DOMContentLoaded', function() {
    initNavigation();
});
```

### Page-Specific Scripts

Create a dedicated file and load it from the template:

```php
<?php $this->start('scripts') ?>
<script src="/Assets/js/myfeature.js"></script>
<?php $this->end() ?>
```

Wrap in an IIFE to avoid global scope pollution:

```javascript
(function() {
    'use strict';

    function init() { /* ... */ }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        init();
    }
})();
```

Use event delegation for dynamic content:

```javascript
document.addEventListener('click', function(e) {
    if (e.target.matches('.delete-btn')) {
        handleDelete(e.target);
    }
});
```

### CDN Libraries

External libraries load from CDNs:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
<script src="/Assets/js/analytics.js" defer></script>
```

## Cache Busting

For production, append a version query string to force reloads when files change:

```php
$cssUrl = '/Assets/css/app.css?v=' . filemtime('Public/Assets/css/app.css');
```

---

**Related Docs:**
- [URI_TYPE_STATIC.md](../architecture/URI_TYPE_STATIC.md) - How static files are served
- [STATIC_FILE_SERVING_SECURITY.md](../architecture/STATIC_FILE_SERVING_SECURITY.md) - Security considerations
