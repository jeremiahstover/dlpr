# Asset Pipeline

The asset pipeline handles CSS, JavaScript, and image files. ML2 uses a simple static file approach with theme-based CSS loading.

## Overview

Assets are served directly from the `Public/Assets/` directory without build tools or bundling:

- **Static Files**: Direct file serving, no compilation step
- **Theme-Based CSS**: Different themes load different CSS files
- **CDN Libraries**: External libraries loaded from CDNs
- **Role-Based CSS**: Additional CSS per user role

## Directory Structure

```
Public/Assets/
├── css/
│   ├── ml2-theme.css          # Primary theme (default)
│   ├── original-theme.css     # Original theme
│   ├── crgslst-theme.css      # Craigslist-style theme
│   ├── dark-theme.css         # Dark mode variant
│   ├── admin.css              # Admin-specific styles
│   ├── manager.css            # Manager-specific styles
│   ├── verse.picker.css       # Verse picker component
│   └── verse.picker.dark.css  # Verse picker dark mode
├── js/
│   ├── site.js                # Global site JavaScript
│   ├── study-form.js          # Study form functionality
│   ├── collection-editor.js   # Collection editor
│   ├── verse.picker.js        # Verse picker component
│   └── custom.js              # Custom/user scripts
├── img/
│   ├── favicon.ico
│   ├── memorize_bg.jpg
│   └── ...
└── data/
    └── ...                    # JSON data files
```

## CSS Architecture

### Theme Loading

Themes are loaded based on the `app:theme` config value:

```php
// In header partial
$theme = $config->get('app:theme', 'ml2');
$themeCss = "/Assets/css/{$theme}-theme.css";
```

### CSS File Structure

```css
/* ml2-theme.css */

/* 1. CSS Variables (Design Tokens) */
:root {
    --primary: #0055ff;
    --primary-hover: #0044cc;
    --background: #ffffff;
    --text: #1a1a1a;
    --border: #e0e0e0;
    --shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* 2. PicoCSS Overrides */
:root {
    --pico-primary: var(--primary);
    --pico-primary-hover: var(--primary-hover);
}

/* 3. Component Styles */
.study-card {
    background: var(--background);
    border: 1px solid var(--border);
    box-shadow: var(--shadow);
}

/* 4. Utility Classes */
.text-center { text-align: center; }
.mt-4 { margin-top: 2rem; }
```

### Role-Based CSS

Additional CSS files load based on user role:

```php
// In header partial
if ($userRole) {
    $roleCssFile = '/Assets/css/' . $userRole . '.css';
    if (file_exists($roleCssPath)) {
        echo '<link rel="stylesheet" href="' . $roleCssFile . '">';
    }
}
```

Example: An admin user loads both `ml2-theme.css` and `admin.css`.

## JavaScript Loading

### Global Scripts

Loaded in header/footer for all pages:

```html
<!-- In header -->
<script src="/Assets/js/site.js"></script>
```

### Page-Specific Scripts

Loaded only on specific pages:

```php
// In study form template
<script src="/Assets/js/study-form.js"></script>
```

### CDN Libraries

External libraries loaded from CDNs:

```html
<!-- PicoCSS -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">

<!-- Font Awesome -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
```

## Creating a New Theme

### Step 1: Create CSS File

```bash
touch Public/Assets/css/mytheme-theme.css
```

### Step 2: Define Variables

```css
/* mytheme-theme.css */
:root {
    /* Brand colors */
    --primary: #your-brand-color;
    --primary-hover: #darker-version;
    
    /* Semantic colors */
    --success: #28a745;
    --warning: #ffc107;
    --error: #dc3545;
    
    /* Layout */
    --max-width: 1200px;
    --spacing: 1rem;
    
    /* Typography */
    --font-family: 'Your Font', sans-serif;
    --font-size: 16px;
}
```

### Step 3: Override PicoCSS

```css
/* Override PicoCSS variables */
:root {
    --pico-primary: var(--primary);
    --pico-primary-hover: var(--primary-hover);
    --pico-font-family: var(--font-family);
}
```

### Step 4: Add Component Styles

```css
/* Custom components */
.mytheme-card {
    border-radius: 8px;
    background: white;
    box-shadow: var(--shadow);
}
```

### Step 5: Update Config

```json
{
    "app": {
        "theme": "mytheme"
    }
}
```

## Adding JavaScript

### Global Script

Add to `Public/Assets/js/site.js`:

```javascript
// site.js - Available on all pages

document.addEventListener('DOMContentLoaded', function() {
    // Initialize global functionality
    initNavigation();
    initNotifications();
});

function initNavigation() {
    // Mobile menu toggle
    const menuToggle = document.querySelector('.menu-toggle');
    if (menuToggle) {
        menuToggle.addEventListener('click', toggleMenu);
    }
}
```

### Page-Specific Script

Create `Public/Assets/js/myfeature.js`:

```javascript
// myfeature.js - Only loaded on specific pages

(function() {
    'use strict';
    
    function init() {
        // Feature initialization
    }
    
    // Auto-init when DOM is ready
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        init();
    }
})();
```

Load in template:

```php
<?php $this->start('scripts') ?>
<script src="/Assets/js/myfeature.js"></script>
<?php $this->end() ?>
```

## Best Practices

### CSS

1. **Use CSS Variables**: Define colors/spacing in `:root`
2. **Follow BEM Naming**: `.block__element--modifier`
3. **Mobile First**: Base styles for mobile, media queries for desktop
4. **Avoid !important**: Use specificity instead

```css
/* Good */
.study-card__title {
    font-size: 1.25rem;
}

@media (min-width: 768px) {
    .study-card__title {
        font-size: 1.5rem;
    }
}

/* Bad */
.study-card h3 {
    font-size: 1.5rem !important;
}
```

### JavaScript

1. **Use IIFE or Modules**: Avoid global scope pollution
2. **Feature Detection**: Check before using APIs
3. **Event Delegation**: For dynamic content

```javascript
// Good - Event delegation
document.addEventListener('click', function(e) {
    if (e.target.matches('.delete-btn')) {
        handleDelete(e.target);
    }
});

// Bad - Individual listeners
document.querySelectorAll('.delete-btn').forEach(btn => {
    btn.addEventListener('click', handleDelete);
});
```

### Performance

1. **Minimize HTTP Requests**: Combine small CSS/JS files
2. **Use CDN for Libraries**: Cache benefits
3. **Defer Non-Critical JS**: Load after page render
4. **Optimize Images**: Compress, use appropriate formats

```html
<!-- Deferred loading -->
<script src="/Assets/js/analytics.js" defer></script>

<!-- Async loading -->
<script src="/Assets/js/chat-widget.js" async></script>
```

## Image Assets

### Supported Formats

- **SVG**: Icons, logos (scalable)
- **PNG**: Images with transparency
- **JPG**: Photographs
- **ICO**: Favicons

### Favicon

```html
<link rel="icon" type="image/x-icon" href="/Assets/img/favicon.ico">
```

### Background Images

```css
.hero {
    background-image: url('/Assets/img/hero-bg.jpg');
    background-size: cover;
}
```

## Versioning / Cache Busting

For production, add version query strings:

```php
$cssUrl = '/Assets/css/ml2-theme.css?v=' . filemtime('Public/Assets/css/ml2-theme.css');
```

This forces browsers to reload when files change.

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
