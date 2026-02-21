# Theming Guide

This guide explains how to create, customize, and manage themes in the Application.

## Overview

Application uses a simple theme system based on CSS files. Themes control the visual appearance of the application including colors, typography, spacing, and layout.

**Key Concepts:**
- **Theme CSS Files**: Located in `Public/Assets/css/`
- **Theme Selection**: Configured via `app:theme` config value
- **PicoCSS Base**: All themes extend PicoCSS framework
- **Role-Based Overrides**: Additional CSS per user role (admin, manager)

## Available Themes

| Theme | File | Description |
|-------|------|-------------|
| **application** | `default-theme.css` | Modern, clean design (default) |
| **original** | `original-theme.css` | Classic Application appearance |
| **crgslst** | `crgslst-theme.css` | Craigslist-style minimal design |
| **dark** | `dark-theme.css` | Dark mode variant |

## Theme Structure

### CSS File Organization

```css
/* 1. CSS Variables (Design Tokens) */
:root {
    --primary: #0055ff;
    --primary-hover: #0044cc;
    --background: #ffffff;
    --text: #1a1a1a;
    --border: #e0e0e0;
    --shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* 2. PicoCSS Variable Overrides */
:root {
    --pico-primary: var(--primary);
    --pico-primary-hover: var(--primary-hover);
    --pico-background-color: var(--background);
    --pico-color: var(--text);
}

/* 3. Custom Elements */
brand, menu, container, toolbar {
    display: block;
}

/* 4. Component Styles */
.study-card {
    background: var(--background);
    border: 1px solid var(--border);
    box-shadow: var(--shadow);
}

/* 5. Utility Classes */
.text-center { text-align: center; }
.mt-4 { margin-top: 2rem; }
```

### Custom Semantic Elements

Application uses custom HTML elements for layout:

```css
/* Brand/Logo area */
brand {
    display: block;
    font-size: 1.25rem;
    font-weight: 600;
}

brand > a {
    display: flex;
    align-items: center;
    gap: 1rem;
    text-decoration: none;
}

/* Navigation menu */
menu {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
}

menu > a {
    margin-left: 1em;
    text-decoration: none;
    color: black;
}

/* Content container */
container {
    display: block;
    max-width: 1024px;
    margin: 0 auto;
    padding: 0 1rem;
}

/* Action toolbar */
toolbar {
    display: block;
    padding: 1rem 0;
}
```

## Creating a New Theme

### Step 1: Create CSS File

Create a new file in `Public/Assets/css/`:

```bash
# Name format: {theme-name}-theme.css
touch Public/Assets/css/mytheme-theme.css
```

### Step 2: Define CSS Variables

Start with the core design tokens:

```css
/* mytheme-theme.css */

:root {
    /* Brand Colors */
    --primary: #your-primary-color;
    --primary-hover: #your-primary-hover;
    --secondary: #your-secondary-color;
    
    /* Background Colors */
    --background: #ffffff;
    --background-alt: #f8f9fa;
    --background-card: #ffffff;
    
    /* Text Colors */
    --text: #1a1a1a;
    --text-muted: #6c757d;
    --text-inverse: #ffffff;
    
    /* Border & Shadow */
    --border: #e0e0e0;
    --border-radius: 0.5rem;
    --shadow: 0 2px 8px rgba(0,0,0,0.1);
    
    /* Typography */
    --font-sans: system-ui, -apple-system, sans-serif;
    --font-serif: Georgia, serif;
    
    /* Spacing */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 2rem;
    --spacing-xl: 4rem;
}
```

### Step 3: Override PicoCSS Variables

Map your design tokens to PicoCSS:

```css
:root {
    --pico-primary: var(--primary);
    --pico-primary-hover: var(--primary-hover);
    --pico-primary-background: var(--primary);
    --pico-primary-inverse: var(--text-inverse);
    
    --pico-background-color: var(--background);
    --pico-card-background-color: var(--background-card);
    --pico-card-sectioning-background-color: var(--background-alt);
    
    --pico-color: var(--text);
    --pico-muted-color: var(--text-muted);
    
    --pico-border-radius: var(--border-radius);
    --pico-box-shadow: var(--shadow);
    
    --pico-font-family: var(--font-sans);
    --pico-font-size: 16px;
    --pico-line-height: 1.6;
}
```

### Step 4: Add Custom Styles

Style custom elements and components:

```css
/* Background image (optional) */
body::before {
    content: "";
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-image: url('/Assets/img/your-bg.jpg');
    background-size: cover;
    background-position: center;
    z-index: -1;
    filter: blur(8px);
    transform: scale(1.05);
}

/* Header styling */
body > header {
    background-color: rgba(255, 255, 255, 0.9);
    padding: 1rem;
}

/* Study cards */
.study-card {
    background: var(--background-card);
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    padding: var(--spacing-md);
    margin-bottom: var(--spacing-md);
}

/* Call-to-action buttons */
.btn-cta {
    background-color: var(--btn-cta-color);
    color: var(--text-inverse);
}
```

### Step 5: Activate Theme

Update the configuration:

```bash
# Via config or admin interface
app:theme = "mytheme"
```

## Customizing Existing Themes

### Override Specific Styles

Create role-specific CSS for targeted overrides:

```css
/* Public/Assets/css/custom-overrides.css */

/* Override primary color */
:root {
    --primary: #ff6600;
    --primary-hover: #e55a00;
}

/* Custom button styles */
.btn-special {
    background: linear-gradient(135deg, var(--primary), var(--secondary));
}
```

### Modify Theme Files Directly

Edit existing theme files:

```bash
# Edit the active theme
vim Public/Assets/css/default-theme.css
```

**Note**: Changes apply immediately (no build step required).

## Role-Based Theming

### Admin Styles

Create `Public/Assets/css/admin.css`:

```css
/* Admin-specific styles */
.admin-panel {
    border-left: 4px solid var(--primary);
}

.admin-badge {
    background: #dc3545;
    color: white;
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
    font-size: 0.75rem;
}
```

### Manager Styles

Create `Public/Assets/css/manager.css`:

```css
/* Manager-specific styles */
.manager-tools {
    background: #f8f9fa;
    border: 1px solid var(--border);
}
```

Role CSS loads automatically when a user with that role is logged in.

## Dark Mode Themes

### Creating a Dark Theme

```css
/* dark-theme.css */

:root {
    /* Invert colors */
    --background: #1a1a1a;
    --background-alt: #2d2d2d;
    --background-card: #2d2d2d;
    
    --text: #e0e0e0;
    --text-muted: #a0a0a0;
    --text-inverse: #1a1a1a;
    
    --border: #404040;
    --shadow: 0 2px 8px rgba(0,0,0,0.3);
    
    /* Adjust primary for dark backgrounds */
    --primary: #6699ff;
    --primary-hover: #88bbff;
}
```

### Component-Specific Dark Mode

```css
/* verse.picker.dark.css */

:root {
    --vp-background: #2d2d2d;
    --vp-text: #e0e0e0;
    --vp-border: #404040;
    --vp-primary: #6699ff;
}
```

## Best Practices

### CSS Variable Naming

```css
/* Good: Semantic names */
--color-primary
--color-error
--spacing-large

/* Avoid: Presentational names */
--blue
--red
--big-space
```

### Responsive Design

```css
/* Mobile-first approach */
container {
    padding: 0 0.5rem;
}

@media (min-width: 768px) {
    container {
        padding: 0 1rem;
    }
}

@media (min-width: 1024px) {
    container {
        padding: 0 2rem;
    }
}
```

### Performance

```css
/* Use transform for animations */
.card:hover {
    transform: translateY(-2px);
    /* Avoid: margin-top: -2px; (causes reflow) */
}

/* Contain paint for complex elements */
.study-card {
    contain: layout style paint;
}
```

### Accessibility

```css
/* Respect user preferences */
@media (prefers-reduced-motion: reduce) {
    * {
        animation-duration: 0.01ms !important;
        transition-duration: 0.01ms !important;
    }
}

/* Focus indicators */
button:focus-visible,
a:focus-visible {
    outline: 2px solid var(--primary);
    outline-offset: 2px;
}

/* Color contrast */
.text-muted {
    color: var(--text-muted);
    /* Ensure 4.5:1 contrast ratio with background */
}
```

## Theme Testing

### Visual Regression

Test theme changes across pages:

```bash
# 1. Clear browser cache
# 2. Test key pages:
#    - Home page
#    - Study list
#    - Study detail
#    - Forms
#    - Admin panels
```

### Cross-Browser Testing

Verify themes in:
- Chrome/Edge (Chromium)
- Firefox
- Safari
- Mobile browsers

### Accessibility Testing

```bash
# Check contrast ratios
# Verify keyboard navigation
# Test with screen readers
```

## Troubleshooting

### Theme Not Loading

1. Check file exists: `Public/Assets/css/{theme}-theme.css`
2. Verify config: `app:theme` value matches filename prefix
3. Clear browser cache
4. Check file permissions

### CSS Variables Not Working

1. Verify variable definition in `:root`
2. Check for typos in variable names
3. Ensure proper `var()` syntax: `var(--variable-name)`

### PicoCSS Overrides Not Applied

1. Load theme CSS **after** PicoCSS
2. Use same specificity or higher
3. Check for CSS syntax errors

## Related Documentation

- [Asset Pipeline](./ASSET_PIPELINE.md) - CSS/JS loading
- [Partials and Themes](../architecture/PARTIALS_AND_THEMES.md) - Template system
- [Config System](../architecture/CONFIG_SYSTEM.md) - Theme configuration
