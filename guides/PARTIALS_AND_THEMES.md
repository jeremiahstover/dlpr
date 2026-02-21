# Partials and Themes

The Partials system provides a themeable, reusable component architecture for the ML2 presentation layer. Partials are PHP classes that render specific page sections (header, menu, notifications, footer) with support for multiple visual themes.

## Overview

Partials solve two problems:
1. **Code Reuse** - Common page sections are defined once and reused across all pages
2. **Theming** - Multiple visual themes can coexist, with runtime theme switching

## Architecture

```
App/Presentation/Partials/
├── HeaderPartial.php           # Renders <head> and opening <body>
├── MenuPartial.php             # Renders navigation menu
├── NotificationsPartial.php    # Renders notification area
├── FooterPartial.php           # Renders closing </body> and </html>
└── theme/                      # Theme directory
    ├── original/               # Default theme
    │   ├── header.php
    │   ├── menu.php
    │   ├── notifications.php
    │   └── footer.php
    ├── ml2/                    # ML2 theme
    │   ├── header.php
    │   ├── menu.php
    │   ├── notifications.php
    │   └── footer.php
    ├── dark/                   # Dark theme
    └── crgslst/                # Custom theme
```

## How Partials Work

### Rendering Flow

1. **Template calls layout**: `<?php $this->layout('layout') ?>`
2. **Layout loads partials**: `layout.php` includes partial classes
3. **Partials render theme files**: Each partial loads its theme-specific template
4. **Theme fallback**: If theme doesn't exist, falls back to `original`

### Partial Classes

Each partial is a static class with a `render()` method:

```php
namespace MemorizeLive\App\Presentation\Partials;

class HeaderPartial {
    public static function render($theme, $title = 'MemorizeLive', $loadVersePicker = false, $userRole = null) {
        // Sanitize inputs
        $theme = htmlspecialchars(basename($theme), ENT_QUOTES | ENT_HTML5, 'UTF-8');
        
        // Try requested theme first
        $filename = __DIR__ . '/theme/' . $theme . '/header.php';
        if (file_exists($filename)) {
            include($filename);
        } else {
            // Fallback to original theme
            include(__DIR__ . '/theme/original/header.php');
        }
        return $body ?? '';
    }
}
```

## Theme System

### Theme Structure

Each theme is a directory containing 4 template files:

```
theme/{theme_name}/
├── header.php         # HTML <head> section + opening <body>
├── menu.php           # Navigation menu HTML
├── notifications.php  # Notification container HTML
└── footer.php         # Closing tags + scripts
```

### Creating a New Theme

1. **Create theme directory**:
   ```bash
   mkdir App/Presentation/Partials/theme/mytheme
   ```

2. **Copy from original**:
   ```bash
   cp App/Presentation/Partials/theme/original/* \
      App/Presentation/Partials/theme/mytheme/
   ```

3. **Customize templates**:
   - Edit CSS links in `header.php`
   - Modify menu structure in `menu.php`
   - Change notification styling in `notifications.php`
   - Update footer content in `footer.php`

4. **Activate theme**:
   Theme is set via user config or application default:
   ```php
   $theme = $user['config']['theme'] ?? 'ml2';
   ```

### Theme Template Variables

Each template receives specific variables:

**header.php**:
- `$theme` - Theme name
- `$title` - Page title
- `$loadVersePicker` - Whether to load verse picker assets
- `$userRole` - User's role (for role-specific CSS)

**menu.php**:
- `$theme` - Theme name
- `$menuItems` - Array of menu items with `url`, `label`, `is_active`
- `$currentUri` - Current request URI

**notifications.php**:
- `$theme` - Theme name

**footer.php**:
- `$theme` - Theme name
- Additional vars via `...$vars` parameter

## Available Partials

### HeaderPartial

Renders the HTML head section and opening body tag.

```php
use MemorizeLive\App\Presentation\Partials\HeaderPartial;

echo HeaderPartial::render(
    theme: 'ml2',
    title: 'My Page Title',
    loadVersePicker: true,
    userRole: 'admin'
);
```

**Features**:
- Pico CSS framework loading
- Theme-specific CSS (`ml2-theme.css`)
- Role-specific CSS (`admin.css`, `user.css`)
- Verse picker assets (conditional)
- Font Awesome icons

### MenuPartial

Renders the navigation menu with active state highlighting.

```php
use MemorizeLive\App\Presentation\Partials\MenuPartial;

echo MenuPartial::render(
    theme: 'ml2',
    menu: $menuService,
    currentUri: '/studies'
);
```

**Features**:
- Active item highlighting based on current URI
- Menu override support (user-customized menus)
- Alias resolution (canonical URLs vs aliases)
- Nested menu support

### NotificationsPartial

Renders the notification/alert container.

```php
use MemorizeLive\App\Presentation\Partials\NotificationsPartial;

echo NotificationsPartial::render('ml2');
```

### FooterPartial

Renders the page footer and closing tags.

```php
use MemorizeLive\App\Presentation\Partials\FooterPartial;

echo FooterPartial::render('ml2', ['logoUrl' => '/Assets/img/logo.png']);
```

## Creating Custom Partials

1. **Create the partial class**:
   ```php
   <?php
   namespace MemorizeLive\App\Presentation\Partials;
   
   class SidebarPartial {
       public static function render($theme, array $widgets): string {
           $themeDir = __DIR__ . '/theme';
           $filename = $themeDir . '/' . basename($theme) . '/sidebar.php';
           
           if (!file_exists($filename)) {
               $filename = $themeDir . '/original/sidebar.php';
           }
           
           include($filename);
           return $body ?? '';
       }
   }
   ```

2. **Create theme templates**:
   ```php
   <!-- theme/original/sidebar.php -->
   <aside class="sidebar">
       <?php foreach ($widgets as $widget): ?>
           <div class="widget">
               <h3><?= htmlspecialchars($widget['title']) ?></h3>
               <p><?= htmlspecialchars($widget['content']) ?></p>
           </div>
       <?php endforeach; ?>
   </aside>
   <?php $body = ob_get_clean(); ?>
   ```

3. **Use in layout**:
   ```php
   <?php
   require_once __DIR__ . '/Partials/SidebarPartial.php';
   use MemorizeLive\App\Presentation\Partials\SidebarPartial;
   ?>
   
   <?= SidebarPartial::render($theme, $widgets) ?>
   ```

## Best Practices

1. **Always sanitize inputs**: Use `htmlspecialchars()` and `basename()` on theme names
2. **Provide fallback**: Always fall back to `original` theme if requested theme missing
3. **Log warnings**: Log when falling back to help debug theme issues
4. **Keep templates simple**: Templates should display data, not process it
5. **Use output buffering**: Capture template output with `ob_get_clean()`

## Integration with Layout

The `layout.php` file ties all partials together:

```php
<?php
require_once __DIR__ . '/Partials/HeaderPartial.php';
require_once __DIR__ . '/Partials/MenuPartial.php';
require_once __DIR__ . '/Partials/NotificationsPartial.php';
require_once __DIR__ . '/Partials/FooterPartial.php';

use MemorizeLive\App\Presentation\Partials\HeaderPartial;
use MemorizeLive\App\Presentation\Partials\MenuPartial;
use MemorizeLive\App\Presentation\Partials\NotificationsPartial;
use MemorizeLive\App\Presentation\Partials\FooterPartial;

$theme = $this->data['theme'] ?? 'ml2';
$title = $this->data['title'] ?? 'MemorizeLive';
$loadVersePicker = $this->data['loadVersePicker'] ?? false;
$userRole = $this->data['userRole'] ?? null;
$currentUri = $this->data['currentUri'] ?? '/';
?>

<?= HeaderPartial::render($theme, $title, $loadVersePicker, $userRole) ?>
<?= MenuPartial::render($theme, $this->data['menu'], $currentUri) ?>
<?= NotificationsPartial::render($theme) ?>

<main class="container">
    <?= $this->section('content') ?>
</main>

<?= FooterPartial::render($theme) ?>
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
