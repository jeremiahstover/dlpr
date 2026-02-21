# Guide: Adding New Interface Levels

Interface levels provide different UI experiences for different user types (focused, normal, power, admin).

## Overview

Interface levels map user interface preference to template resolution:
- User's `interface` field (0-9) determines level
- Presenter.php resolves template paths with fallback
- Templates can be interface-specific or use generic fallback

## Default Interface Levels

| Value | Name | Purpose |
|-------|------|---------|
| 0 | guest | Guest/unauthenticated users |
| 1 | focused | Simplified, large elements |
| 2 | normal | Balanced, standard UI |
| 3 | power | Advanced, compact, more options |
| 4 | programmer | Programmer-specific view |
| 8 | manager | Manager-level interface (not exposed in UI) |
| 9 | admin | Administrative view |

**Note**: Level 0 (guest) is for unauthenticated users. Level 8 (manager) exists in the interface map but is not exposed in user settings due to lack of approval mechanism.

## Step 1: Update Presenter.php

**File:** `App/Presentation/Presenter.php`

Interface level mapping is handled internally in `getUserInterfaceLevel()` method:

```php
/**
 * Get user's interface level from session
 * @return string Interface level name (guest, focused, normal, power, programmer, manager, admin) or null
 */
private function getUserInterfaceLevel(): ?string {
    // Check if user is authenticated
    if (!$this->session->isAuthenticated()) {
        return 'guest';
    }

    // Get interface value from authenticated user's session data
    $authData = $this->session->getAuthData();
    $interfaceValue = $authData['interface'] ?? 0;

    // Map interface flag to level name
    $levelMap = [
        0 => 'guest',        // Unauthenticated users
        1 => 'focused',      // Simplified UI
        2 => 'normal',       // Standard UI
        3 => 'power',        // Advanced UI
        4 => 'programmer',   // Programmer-specific view
        8 => 'manager',      // Manager level (not exposed in UI)
        9 => 'admin',        // Administrative view
    ];

    return $levelMap[$interfaceValue] ?? null;
}
```

Add directory support in `resolveTemplate()`:

```php
/**
 * Resolve template path with interface-level fallback
 * @param string $template Template name (e.g., 'Studies/list')
 * @return string Resolved template name
 */
private function resolveTemplate(string $template): string {
    $interfaceLevel = $this->getUserInterfaceLevel();

    // If no interface level, use default template
    if (!$interfaceLevel) {
        return $template;
    }

    // Parse template path (e.g., 'Studies/list' -> ['Studies', 'list'])
    $parts = explode('/', $template);

    // Only apply interface resolution to certain directories
    // This prevents breaking other templates like Auth/login, Errors/error, etc.
    if (count($parts) >= 2 && in_array($parts[0], ['Studies', 'Schedules', 'Notifications'])) {
        // Try interface-specific path first (e.g., 'Studies/normal/list')
        $interfaceTemplate = $parts[0] . '/' . $interfaceLevel . '/' . implode('/', array_slice($parts, 1));
        $interfaceTemplatePath = $this->templatesPath . '/' . $interfaceTemplate . '.php';

        if (file_exists($interfaceTemplatePath)) {
            return $interfaceTemplate;
        }
    }

    // Fall back to original template
    return $template;
}
```

**Important Notes:**
- `getUserInterfaceLevel()` returns a string (level name) or null, not an int
- Interface resolution only applies to `Studies`, `Schedules`, and `Notifications` directories
- Guest users (level 0) are handled by the authentication check in `getUserInterfaceLevel()`

## Step 2: Create Template Directory Structure

```
App/Presentation/
├── NewFeature/              # New directory
│   ├── guest/              # Level 0 (unauthenticated)
│   │   └── list.php
│   ├── focused/            # Level 1
│   │   └── list.php
│   ├── normal/             # Level 2 (standard)
│   │   └── list.php
│   ├── power/              # Level 3
│   │   └── list.php
│   ├── programmer/         # Level 4
│   │   └── list.php
│   ├── manager/            # Level 8 (not exposed in UI)
│   │   └── list.php
│   ├── admin/              # Level 9
│   │   └── list.php
│   └── list.php            # Fallback (required)
```

**Important**: Interface-specific templates currently only apply to `Studies`, `Schedules`, and `Notifications` features. Other directories (like `Auth`, `User`, `Collections`, `Errors`) use the generic fallback template.

## Step 3: Create Interface-Specific Templates

### Focused Template (simplified)
```php
<!-- App/Presentation/NewFeature/focused/list.php -->
<?php $this->layout('layout', ['title' => 'My Studies']) ?>
<div class="section">
    <h1 class="title is-1">My Studies</h1>
    <?php foreach ($studies as $study): ?>
        <div class="box">
            <h2 class="title is-4"><?= $this->e($study['title']) ?></h2>
            <p><?= $this->e($study['reference']) ?></p>
        </div>
    <?php endforeach; ?>
</div>
```

### Power Template (compact, advanced)
```php
<!-- App/Presentation/NewFeature/power/list.php -->
<?php $this->layout('layout', ['title' => 'Studies']) ?>
<table class="table is-striped is-fullwidth">
    <thead>
        <tr>
            <th>Title</th>
            <th>Type</th>
            <th>Reference</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        <?php foreach ($studies as $study): ?>
            <tr>
                <td><?= $this->e($study['title']) ?></td>
                <td><?= $this->e($study['type']) ?></td>
                <td><?= $this->e($study['reference']) ?></td>
                <td>
                    <a href="/study/view?id=<?= $study['id'] ?>">View</a>
                    <a href="/study/edit?id=<?= $study['id'] ?>">Edit</a>
                </td>
            </tr>
        <?php endforeach; ?>
    </tbody>
</table>
```

## Step 4: Add Level to User Settings

**File:** `App/Presentation/User/edit-config.php`

The interface level dropdown is populated dynamically via `$interface_options`:

```php
<div class="field">
    <label class="label">UI Level</label>
    <div class="control">
        <div class="select is-fullwidth">
            <select name="interface_level">
                <?php foreach ($interface_options ?? [] as $level => $label): ?>
                    <option value="<?= htmlspecialchars((string)$level) ?>" <?= ($interface_level ?? 0) == $level ? 'selected' : '' ?>>
                        <?= htmlspecialchars($label) ?>
                    </option>
                <?php endforeach; ?>
            </select>
        </div>
    </div>
    <p class="help">User interface level determines available features.</p>
</div>
```

**Controller Implementation** (`App/Routes/Controllers/ProfileController.php`):

```php
private function buildFilteredInterfaceOptions(array $interfaceMap, int $currentInterfaceLevel, string $userEmail): array
{
    $adminsConfig = $this->config->get('admins', '');

    $interfaceOptions = [];
    if (is_array($interfaceMap)) {
        foreach ($interfaceMap as $level => $name) {
            $normalizedLevel = (int)$level;

            // Skip guest - users can't downgrade to guest
            if ($normalizedLevel === 0) {
                continue;
            }

            // Skip manager - no approval mechanism exists
            if ($normalizedLevel === 8) {
                continue;
            }

            // Only include admin if user's email is in admins config
            if ($normalizedLevel === 9) {
                if (!$this->userService->isAdminEmail($userEmail, $adminsConfig)) {
                    continue;
                }
            }

            // Include levels that pass the restrictions
            $label = ucwords(str_replace('_', ' ', (string)$name));
            $interfaceOptions[$normalizedLevel] = $label;
        }

        ksort($interfaceOptions);
    }

    return $interfaceOptions;
}
```

**Available Options After Filtering:**
- **Regular users**: 1 (Focused), 2 (Normal), 3 (Power), 4 (Programmer)
- **Admin users** (email in admins config): 1 (Focused), 2 (Normal), 3 (Power), 4 (Programmer), 9 (Admin)
- **Always excluded**: 0 (Guest) - users cannot downgrade to guest
- **Always excluded**: 8 (Manager) - no approval mechanism exists

## Step 5: Handle Level in Controller

Controllers don't need changes - template resolution is automatic. But if level-specific logic is needed:

```php
class NewFeatureController {
    public function list($app, $request) {
        $level = $_SESSION['auth']['interface'] ?? 2;
        
        // Level-specific data adjustments
        if ($level === 1) {
            // Focused: fewer items, simpler data
        } elseif ($level === 3) {
            // Power: include all metadata
        }
        
        return [
            'template' => 'NewFeature/list',
            'data' => ['studies' => $studies]
        ];
    }
}
```

## Testing Checklist

- [ ] Interface map includes all levels in Presenter.php
- [ ] User can set interface level in profile settings
- [ ] Interface options are filtered correctly (guest and manager excluded, admin restricted)
- [ ] Level stored in session after login
- [ ] Correct template loads for each level (Studies, Schedules, Notifications only)
- [ ] Fallback template loads when interface-specific missing
- [ ] Guest users see appropriate templates
- [ ] Interface resolution does not affect other directories (Auth, User, Collections, Errors)
- [ ] Admin users can access admin-level templates (if email in admins config)

## Fallback Behavior

```
Request: Studies/list (interface=normal/2)
    ↓
Try: Studies/normal/list.php (interface-specific)
    ↓
If not found: Studies/list.php (generic fallback)
    ↓
Returns: Either Studies/normal/list.php or Studies/list.php
```

**Key Points:**
- If user is not authenticated, `getUserInterfaceLevel()` returns 'guest'
- If interface-specific template doesn't exist, falls back to generic template
- Interface resolution only applies to `Studies`, `Schedules`, and `Notifications` directories
- All other directories use the generic template directly

## Adding New Level Summary

1. Add level mapping to `$levelMap` array in `getUserInterfaceLevel()` in Presenter.php
2. If adding access control, update `buildFilteredInterfaceOptions()` in ProfileController.php
3. Create directory in `App/Presentation/` for the feature (if needed)
4. Create interface-specific subdirectories and templates (only for Studies, Schedules, Notifications)
5. Update interface_map in config.json.example if needed
6. Test template resolution for all levels
