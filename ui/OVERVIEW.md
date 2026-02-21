# UI Hub Overview

The ML2 UI Hub is built on the **DLPR (Data, Logic, Presentation, Routes)** architecture. The Presentation layer orchestrates three core systems—**Themes**, **Interface Levels**, and **Menus**—to deliver a cohesive, adaptable user experience.

## Architecture Flow

```
User Request
    ↓
Routes Layer (Controller)
    ↓
Logic Layer (Services/Validation)
    ↓
Presentation Layer (Presenter.php)
    ├── Theme Resolution
    ├── Interface Level Detection
    ├── Template Resolution
    └── Menu Assembly
    ↓
Rendered HTML Response
```

## Core Systems

### 1. Theme System
Themes control the visual appearance through CSS and structural partials.

- **Location**: `App/Presentation/Partials/theme/{theme}/`
- **Active Themes**: `ml2`, `original`, `dark`, `crgslst`
- **Fallback**: `ml2` (hardcoded default in `Presenter::resolveTheme()`)
- **Configuration**: `config.app.theme`

Themes provide: `header.php`, `menu.php`, `notifications.php`, `footer.php`

### 2. Interface Level System
Interface levels control feature visibility and template specialization based on user expertise.

- **Mapping**: Database `interface` field (0-9) → level name
- **Resolution**: `Presenter::resolveTemplate()` checks for specialized templates
- **Scope**: Applied to `Studies`, `Schedules`, `Notifications` domains only

| Level | Name | Description |
|-------|------|-------------|
| 0 | guest | Unauthenticated visitors |
| 1 | focused | Minimalist, distraction-free UI |
| 2 | normal | Standard feature set |
| 3 | power | Advanced features |
| 4 | programmer | Developer tools |
| 8 | manager | User management access |
| 9 | admin | Full system control |

### 3. Menu System
Menus adapt to the user's interface level and can be customized via database overrides.

- **Source**: `Data/menus.json` (system defaults)
- **Service**: `MenuService` (override resolution)
- **Storage**: `users.config.menu_overrides` (JSON)
- **Priority**: User override → System default → System fallback

## Integration Points

### Presenter.php Central Role
The `Presenter` class is the orchestration hub:

```php
// 1. Detect interface level from session
$interfaceLevel = $this->getUserInterfaceLevel(); // 'focused', 'normal', etc.

// 2. Resolve theme with validation
$theme = $this->resolveTheme($config['app']['theme'] ?? 'ml2');

// 3. Resolve template with interface fallback
$resolvedTemplate = $this->resolveTemplate('Studies/list');
// Checks: Studies/focused/list.php → Studies/list.php

// 4. Assemble menu via MenuService
$data['menu'] = $this->menu;
```

### Template Resolution Chain
When rendering `Studies/list` for a focused user:

1. **Check**: `App/Presentation/Studies/focused/list.php`
2. **Fallback**: `App/Presentation/Studies/list.php`
3. **Render**: Via Plates template engine

Only these domains support interface-level templates:
- `Studies`
- `Schedules`
- `Notifications`

### Menu Assembly Chain
When generating the navigation menu:

1. **Detect**: User's interface level (e.g., `normal`)
2. **Check**: `users.config.menu_overrides` for valid override
3. **Apply**: Override if valid and visible items exist
4. **Fallback**: `Data/menus.json` default for level
5. **Safety**: Inject Profile menu if missing
6. **Filter**: Return only `visible: true` items

## Configuration References

### Interface Map
```json
{
  "interface_map": {
    "0": "guest",
    "1": "focused",
    "2": "normal",
    "3": "power",
    "4": "programmer",
    "8": "manager",
    "9": "admin"
  }
}
```

### Theme Selection
```json
{
  "app": {
    "theme": "ml2"
  }
}
```

User-specific theme can override via `users.theme` field.

## Security Model

The UI Hub enforces these security invariants:

- **Profile Menu Protection**: The Profile link (`/user`) cannot be hidden or disabled
- **Theme Validation**: Invalid theme names fall back to `ml2`
- **Menu Fallbacks**: Empty or invalid menus automatically fall back to safe defaults
- **Interface Isolation**: Users cannot access templates above their interface level

## Related Documentation

- [THEMES.md](./THEMES.md) - Theme system details
- [INTERFACE_LEVELS.md](./INTERFACE_LEVELS.md) - Interface level documentation
- [MENU_SYSTEM.md](./MENU_SYSTEM.md) - Menu system documentation
- [CRITICAL_MENU_DEFENSES.md](./CRITICAL_MENU_DEFENSES.md) - Security defenses
