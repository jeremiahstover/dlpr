# Custom Autoloader System

This document details the custom autoloader implementation used in the ML2 application, which handles class loading for application code. External dependencies are managed by Composer.

## Overview

The ML2 application uses two autoloaders:
1. **Composer's autoloader** - Handles all external dependencies (including Plates)
2. **Custom autoloader** - Handles application code under `MemorizeLive\App\` namespace

The custom autoloader is registered via `spl_autoload_register()` in `App/App.php` to dynamically load PHP classes when they are first referenced.

## Autoloader Registration

The autoloader is registered in `App/App.php` in the `loadAutoloader()` method:

```php
private function loadAutoloader(): void {
    spl_autoload_register(function ($class) {
        // App code - MemorizeLive\App\*
        if (strpos($class, 'MemorizeLive\\App\\') === 0) {
            $path = str_replace('MemorizeLive\\App\\', '', $class);
            $path = str_replace('\\', '/', $path);
            $file = __DIR__ . '/' . $path . '.php';
            
            if (file_exists($file)) {
                require_once $file;
                return;
            }
        }
    });
}
```

## Application Code Loading

Application classes under the `MemorizeLive\App\` namespace are loaded from the `App/` directory using direct PSR-4 style mapping:

- **Namespace Pattern**: `MemorizeLive\App\{SubNamespace}\{ClassName}`
- **File Path Mapping**: `App/{SubNamespace}/{ClassName}.php`

**Examples:**
```
Class: MemorizeLive\App\Routes\Router
Path:  App/Routes/Router.php

Class: MemorizeLive\App\Logic\Services\StudyService
Path:  App/Logic/Services/StudyService.php

Class: MemorizeLive\App\Data\Repositories\UserRepository
Path:  App/Data/Repositories/UserRepository.php
```

## External Dependencies

External libraries (like League\Plates) are loaded by Composer's autoloader, not the custom autoloader. Composer's autoloader is loaded first in `Public/index.php`:

```php
// Load Composer autoloader for dependencies (like Plates)
require_once __DIR__ . '/../Lib/autoload.php';
```

The custom autoloader only handles the `MemorizeLive\App\` namespace. All other namespaces are handled by Composer.

## Namespace Mapping Strategy

| Namespace Prefix | Handler | Directory |
|-----------------|---------|-----------|
| `MemorizeLive\App\` | Custom autoloader | `App/` |
| `League\Plates\` | Composer autoloader | `Lib/league/plates/src/` |
| All others | Composer autoloader | Various in `Lib/` |

## Class File Discovery Algorithm

When a class is referenced but not yet loaded, the custom autoloader:

1. **Receives the fully qualified class name**
2. **Checks if it starts with `MemorizeLive\App\`**
3. **Converts the namespace to a file path:**
   - Removes the `MemorizeLive\App\` prefix
   - Replaces namespace separators (`\`) with directory separators (`/`)
   - Appends `.php` extension
4. **Constructs the full file path:** `App/{SubNamespace}/{ClassName}.php`
5. **Checks if the file exists**
6. **Requires the file if it exists**

## PSR-4 Compliance

The custom autoloader follows PSR-4 principles for the namespace it handles:

- **Namespace and Directory Structure Alignment**: Class namespaces directly map to directory structures
- **File Name Convention**: Class files are named exactly as the class name with `.php` extension
- **Predictable Loading**: Given a class name, its file location can be determined without configuration

The autoloader is simplified compared to a full PSR-4 implementation because:
- It only handles one namespace prefix
- It doesn't support multiple directory paths per namespace
- It doesn't handle edge cases like trailing namespace separators

## Performance Characteristics

- **First Load**: Minimal file system operations (1 `file_exists()` call)
- **Subsequent Loads**: No overhead (classes cached by PHP)
- **Memory Impact**: Minimal (only loads required files)
- **No Composer overhead** for application classes

## Error Handling

The autoloader silently fails when a class file is not found, allowing PHP's default error handling to take over. This approach:
- Prevents autoloader-specific error messages
- Maintains compatibility with other autoloaders
- Allows PHP to generate appropriate "class not found" errors

## Integration with Composer

The autoloading order ensures proper priority:

1. **Composer's autoloader is registered first** (in `Public/index.php`)
2. **Custom autoloader is registered second** (in `App/App.php` constructor)
3. **Composer handles external dependencies**
4. **Custom autoloader handles application code**

This ensures:
- External libraries are loaded by Composer
- Application classes are loaded by the custom autoloader
- No conflicts between the two autoloaders

## Security Considerations

The autoloader includes implicit security measures:
- **Path Restriction**: Only loads files from the `App/` directory
- **Namespace Validation**: Only processes classes with the `MemorizeLive\App\` prefix
- **File Existence Checking**: Prevents inclusion of non-existent files

These measures prevent:
- Directory traversal attacks through class names
- Loading of arbitrary files
- Namespace pollution from unexpected class loading

---

**See Also:**
- [ENTRY_POINTS.md](ENTRY_POINTS.md) - Application entry points and bootstrapping
- `Public/index.php` - Where Composer autoloader is loaded
- `App/App.php` - Where custom autoloader is registered