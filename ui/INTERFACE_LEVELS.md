# Interface Levels

Interface levels provide a tiered user experience, controlling feature visibility and template specialization based on user expertise and role.

## Level Mapping

The system maps numeric database values to named interface levels:

| Value | Name | Audience | Key Access |
|-------|------|----------|------------|
| 0 | `guest` | Unauthenticated visitors | Public pages only |
| 1 | `focused` | Minimalist users | Core features only |
| 2 | `normal` | Standard users | Full user features |
| 3 | `power` | Power users | Advanced features |
| 4 | `programmer` | Developers | Debug/tools access |
| 8 | `manager` | User managers | User management |
| 9 | `admin` | System administrators | Full system access |

### Configuration
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

## Level Detection

Interface level is determined from authenticated session data:

```php
private function getUserInterfaceLevel(): ?string {
    // Unauthenticated users are 'guest'
    if (!$this->sessionRepository->isAuthenticated()) {
        return 'guest';
    }
    
    $authData = $this->sessionRepository->getAuthData();
    $interfaceValue = $authData['interface'] ?? 0;
    
    $levelMap = [
        0 => 'guest',
        1 => 'focused',
        2 => 'normal',
        3 => 'power',
        4 => 'programmer',
        8 => 'manager',
        9 => 'admin'
    ];
    
    return $levelMap[$interfaceValue] ?? null;
}
```

## Template Resolution

### Scope
Interface-level template resolution applies only to these domains:
- `Studies`
- `Schedules`
- `Notifications`

### Resolution Logic
The `Presenter::resolveTemplate()` method implements the lookup chain:

```php
private function resolveTemplate(string $template): string {
    $interfaceLevel = $this->getUserInterfaceLevel();
    
    if (!$interfaceLevel) {
        return $template;
    }
    
    $parts = explode('/', $template);
    
    // Only resolve for specific domains
    if (count($parts) >= 2 && in_array($parts[0], ['Studies', 'Schedules', 'Notifications'])) {
        // Try interface-specific path first
        $interfaceTemplate = $parts[0] . '/' . $interfaceLevel . '/' . implode('/', array_slice($parts, 1));
        $interfaceTemplatePath = $this->templatesPath . '/' . $interfaceTemplate . '.php';
        
        if (file_exists($interfaceTemplatePath)) {
            return $interfaceTemplate;
        }
    }
    
    // Fall back to base template
    return $template;
}
```

### Resolution Chain Example
Request: `Studies/list` for a focused user

1. Check: `App/Presentation/Studies/focused/list.php`
2. Fallback: `App/Presentation/Studies/list.php`

### Directory Structure
```
App/Presentation/Studies/
├── focused/
│   └── list.php          # Focused variant
├── normal/
│   └── list.php          # Normal variant
├── power/
│   └── list.php          # Power variant
├── list.php              # Base fallback
├── form.php              # Base only
└── view.php              # Base only
```

## Menu Integration

Each interface level has a corresponding menu definition in `Data/menus.json`:

```json
{
  "guest": [
    {"label": "Features", "url": "/features", "pos": "start", "visible": true},
    {"label": "Get Started", "url": "/user/login", "pos": "end", "visible": true}
  ],
  "focused": [
    {"label": "Studies", "url": "/studies", "pos": "start", "visible": true},
    {"label": "Profile", "url": "/user", "pos": "end", "visible": true},
    {"label": "Logout", "url": "/user/logout", "pos": "end", "visible": true}
  ],
  "normal": [
    {"label": "Studies", "url": "/studies", "pos": "start", "visible": true},
    {"label": "Schedules", "url": "/schedules", "pos": "start", "visible": true},
    {"label": "Notifications", "url": "/notifications", "pos": "start", "visible": true},
    {"label": "Collections", "url": "/collections", "pos": "start", "visible": true},
    {"label": "Profile", "url": "/user", "pos": "end", "visible": true},
    {"label": "Logout", "url": "/user/logout", "pos": "end", "visible": true}
  ]
}
```

## Level-Specific Features

### Guest (0)
- Public pages only
- Login/registration access
- Read-only content

### Focused (1)
- Minimal navigation
- Studies primary focus
- Reduced distractions
- No advanced features

### Normal (2)
- Full user feature set
- Schedules, notifications, collections
- Standard menu density

### Power (3)
- Advanced study options
- Batch operations
- Enhanced filtering

### Programmer (4)
- Debug information access
- API inspection tools
- System diagnostics

### Manager (8)
- User management access
- View user listings
- Basic user operations

### Admin (9)
- Full system administration
- User CRUD operations
- System configuration
- All feature access

## Database Schema

The interface level is stored in the `users` table:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    interface INTEGER DEFAULT 2,  -- 0-9 mapping
    -- ... other fields
);
```

## Best Practices

1. **Template Fallbacks**: Always provide a base template; interface-specific variants are optional
2. **Feature Gating**: Use interface level for UI density, not security (enforce security in Logic layer)
3. **Progressive Disclosure**: Higher levels show more options, not different workflows
4. **Consistent Structure**: Keep navigation consistent across levels; vary density, not structure

## Related Files

- `App/Presentation/Presenter.php` - Resolution logic
- `Data/menus.json` - Menu definitions per level
- `Data/config.json.example` - Interface map configuration
