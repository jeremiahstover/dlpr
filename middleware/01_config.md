# Tier 01: ConfigMiddleware

**Purpose**: Load global configuration from `Data/config.json` and make it available to all downstream middleware and controllers.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['config']` | `array` | Complete configuration tree from `config.json` |

## Responsibilities

1. **Instantiate ConfigManager**: Creates a `ConfigManager` instance that handles hierarchical configuration access
2. **Load Configuration**: Reads all values from `Data/config.json` via `ConfigRepository`
3. **Attach to Request**: Adds the complete config array to `$request['config']`

## Implementation

```php
class ConfigMiddleware implements MiddlewareInterface {
    private ConfigManager $configManager;

    public function __construct() {
        $this->configManager = new ConfigManager();
    }

    public function handle(array $request): array {
        $request['config'] = $this->configManager->getAll();
        return $request;
    }
}
```

## ConfigManager Integration

The `ConfigManager` provides hierarchical key access using colon-separated paths:

```php
// Accessing nested config values
$perPage = $configManager->get('pagination:per_page', 20);
$theme = $configManager->get('app:theme', 'ml');
```

When a default is provided and the key doesn't exist:
- The default value is returned
- The nested structure is auto-created
- The config file is automatically persisted

## Configuration Structure

The config array typically contains:

```php
$request['config'] = [
    'app' => [
        'theme' => 'ml',
        'base_url' => 'https://example.com',
    ],
    'encryption' => [
        'key' => '...',
    ],
    'admins' => ['admin@example.com'],
    'interface_map' => [
        '0' => 'guest',
        '1' => 'focused',
        '2' => 'normal',
        '9' => 'admin',
    ],
    'study_types' => ['verse', 'topic', 'reference', 'collection'],
    // ... other configuration
];
```

## Downstream Usage

Middleware tiers that depend on config:

| Middleware | Config Usage |
|-----------|-------------|
| IdentityMiddleware | Reads `interface_map` and `admins` for role determination |
| AccessMiddleware | Reads `resources:*` and `roles:*` for access rules |
| InputValidation | Reads `study_types` and `interface_map` for schema initialization |

Controllers access config via:

```php
$config = $request['config'];
$theme = $config['app']['theme'] ?? 'ml';
$admins = $config['admins'] ?? [];
```

## Error Handling

ConfigMiddleware does not throw exceptions. If `config.json` is missing or unreadable, the `ConfigRepository` handles the error internally and returns an empty array.

## Dependencies

- **ConfigManager**: `/App/Logic/Helpers/ConfigManager.php`
- **ConfigRepository**: `/App/Data/Repositories/ConfigRepository.php`
- **Source File**: `Data/config.json`

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 02: IdentityMiddleware](02_identity.md)
