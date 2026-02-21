# Configuration System

The Configuration System provides a hierarchical, persistent configuration management solution using JSON files. It supports environment-specific configs, auto-populating defaults, and colon-separated key paths.

## Overview

The system consists of two main components:
- **ConfigRepository**: Handles file I/O and caching
- **ConfigManager**: Provides hierarchical access and auto-populating defaults

## Architecture

```
Data/
├── config.json              # Default configuration (fallback)
├── {hostname}.json         # Hostname-specific config (primary)
└── ...

App/
├── Data/
│   └── Repositories/
│       └── ConfigRepository.php    # File operations
└── Logic/
    └── Helpers/
        └── ConfigManager.php       # Access layer
```

## ConfigRepository

Handles low-level configuration file operations.

### File Resolution (Priority Order)

1. `Data/{hostname}.json` - Hostname-specific config
2. `Data/config.json` - Default fallback config

### Key Methods

```php
$configRepo = new ConfigRepository();

// Get all config
$allConfig = $configRepo->getAll();

// Get single value
$value = $configRepo->get('database', []);

// Clear cache (after external changes)
$configRepo->clearCache();
```

## ConfigManager

Provides high-level configuration access with hierarchical keys and auto-populating defaults.

### Colon-Separated Keys

Use colons to access nested values:

```php
$config = new ConfigManager();

// Simple key
$theme = $config->get('app:theme', 'default');

// Nested key
$perPage = $config->get('pagination:per_page', 20);

// Deep nesting
$timeout = $config->get('apis:schedule:timeout', 30);
```

### Auto-Populating Defaults

When you request a key that doesn't exist with a default value:

1. The default is returned
2. The key is created in the nested structure
3. The config file is automatically saved

```php
// First call - key doesn't exist
$value = $config->get('new:feature:enabled', true);
// Returns: true
// Side effect: Creates new.feature.enabled = true in config file

// Second call - key now exists
$value = $config->get('new:feature:enabled', true);
// Returns: true (from saved config)
```

## Configuration Structure

### Standard Sections

```json
{
    "database": {
        "host": "localhost",
        "name": "application",
        "user": "app",
        "pass": "secret"
    },
    "app": {
        "theme": "application",
        "base_url": "https://example.com",
        "debug": false
    },
    "pagination": {
        "per_page": 20
    },
    "encryption": {
        "key": "..."
    },
    "apis": {
        "verseapi": "https://verseapi.memorize.live",
        "scheduleapi": "https://scheduleapi.memorize.live",
        "timeout": 30
    },
    "routes": {
        "admin": null,
        "study_pattern": "#^/studies/(?\\d+)(?:/|$)#"
    },
    "resources": {
        "studies": "studies",
        "collections": "collections"
    },
    "roles": {
        "admin": "admin"
    }
}
```

### Environment-Specific Configs

Create hostname-specific configs for different environments:

```bash
# Development machine
Data/dev-server.json

# Production server
Data/prod-web01.json

# Local development
Data/jeremiah-laptop.json
```

The system automatically uses the hostname-specific file if it exists.

## Usage Examples

### Basic Access

```php
use Application\App\Logic\Helpers\ConfigManager;

$config = new ConfigManager();

// Get with default
$dbConfig = $config->get('database', []);
$theme = $config->get('app:theme', 'default');
```

### In Controllers

```php
class MyController {
    public function __construct(
        private ConfigManager $config
    ) {}
    
    public function action() {
        $perPage = $this->config->get('pagination:per_page', 20);
        // ...
    }
}
```

### Adding New Configuration

```php
// Set a new config value with default
$apiTimeout = $config->get('apis:newservice:timeout', 60);

// This automatically creates:
// {
//     "apis": {
//         "newservice": {
//             "timeout": 60
//         }
//     }
// }
```

## Best Practices

1. **Use colons for nesting**: `app:theme` not `app.theme`
2. **Always provide defaults**: `$config->get('key', 'default')`
3. **Group related settings**: `apis:verseapi`, `apis:scheduleapi`
4. **Use hostname configs**: Isolate environment-specific settings
5. **Don't commit secrets**: Use environment variables for sensitive data

## Security Considerations

### File Permissions

Config files should be readable by web server but not writable:

```bash
chmod 644 Data/config.json
chmod 644 Data/{hostname}.json
```

### Sensitive Data

For sensitive configuration (API keys, passwords):

```php
// config.json
{
    "apis": {
        "mailgun_key": "${MAILGUN_API_KEY}"
    }
}
```

Load from environment variables in application bootstrap.

### Validation

ConfigManager throws exceptions for:
- Missing config files (in development)
- Invalid JSON
- File write failures

## Troubleshooting

### Config Changes Not Reflected

Clear the cache:

```php
$configRepo->clearCache();
```

### Wrong Config File Loading

Check hostname:

```bash
php -r "echo gethostname();"
```

Verify file exists:

```bash
ls -la Data/$(hostname).json
```

### Config File Corrupted

Restore from backup or reset to defaults:

```bash
# Backup first
cp Data/config.json Data/config.json.bak

# Reset
echo '{}' > Data/config.json
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
