# Tier 06: UserContext

**Purpose**: Hydrate the request with user-specific preferences from the database, providing controllers with ready access to timezone, theme, interface level, and configuration.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['user_context']` | `array` | User preferences loaded from database |

## User Context Structure

```php
$request['user_context'] = [
    'id' => 42,
    'email' => 'user@example.com',
    'name' => 'John Doe',
    'timezone' => 'America/New_York',
    'interface' => 2,
    'config' => [
        'theme' => 'default',
        'login_length' => 7,
        'home_route' => 'studies',
        // ... other user config
    ],
    'email_validated' => true,
    'phone_validated' => false,
    'phone' => '+1234567890',
];
```

### Context Fields

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `id` | `int` | `users.id` | User ID |
| `email` | `string` | `users.email` | Email address |
| `name` | `?string` | `users.name` | Display name |
| `timezone` | `?string` | `users.timezone` | IANA timezone identifier |
| `interface` | `int` | `users.interface` | Interface level (0-9) |
| `config` | `array` | `users.config` (JSON) | User configuration object |
| `email_validated` | `bool` | `users.verified` | Email verification status |
| `phone_validated` | `bool` | `users.phone_validated` | Phone verification status |
| `phone` | `?string` | `users.phone` | Phone number |

## Responsibilities

1. **Authentication Check**: Verifies `$request['identity']` exists and is valid
2. **User Lookup**: Loads user record from `UserRepository` by ID
3. **Config Parsing**: Decodes JSON config field into array
4. **Context Assembly**: Builds structured user context array
5. **Request Attachment**: Adds context to `$request['user_context']`

## Implementation

```php
public function handle(array $request): array {
    // Check if user is authenticated
    if (isset($request['identity']) && $request['identity'] instanceof Identity) {
        $userId = $request['identity']->id ?? null;

        if ($userId) {
            // Load user data from database
            $user = $this->userRepository->findById($userId);

            if ($user) {
                // Build user context
                $userContext = [
                    'id' => $user['id'],
                    'email' => $user['email'] ?? null,
                    'name' => $user['name'] ?? null,
                    'timezone' => $user['timezone'] ?? null,
                    'interface' => $user['interface'] ?? 1,
                    'config' => $user['config'] ?? [],
                    'email_validated' => (bool)($user['verified'] ?? false),
                    'phone_validated' => (bool)($user['phone_validated'] ?? false),
                    'phone' => $user['phone'] ?? null,
                ];

                // Attach user context to request
                $request['user_context'] = $userContext;
            }
        }
    }

    return $request;
}
```

## Flow Diagram

```
Request arrives
       │
       ▼
┌───────────────────────────┐
│ Is $request['identity']   │─No──► Return unchanged
│ set and valid?            │
└───────────────────────────┘
       │ Yes
       ▼
┌───────────────────────────┐
│ Extract identity->id      │
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│ Call UserRepository::     │
│ findById($userId)         │
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│ User found?               │─No──► Return unchanged
└───────────────────────────┘
       │ Yes
       ▼
┌───────────────────────────┐
│ Decode users.config JSON  │
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│ Build $userContext array  │
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│ Set $request['user_      │
│ context']                 │
└───────────────────────────┘
       │
       ▼
   Continue
```

## Unauthenticated Requests

For unauthenticated requests (no identity or null identity):

- No database lookup is performed
- `$request['user_context']` is not set
- Controllers must check `isset($request['user_context'])` before accessing

## Database Schema

The `users` table structure:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    email TEXT NOT NULL,
    name TEXT,
    timezone TEXT,
    interface INTEGER DEFAULT 1,
    config TEXT,  -- JSON string
    verified INTEGER DEFAULT 0,
    phone_validated INTEGER DEFAULT 0,
    phone TEXT,
    -- ... other fields
);
```

## Config Field

The `config` field stores JSON-encoded user preferences:

```json
{
    "theme": "default",
    "login_length": 7,
    "home_route": "dashboard",
    "suppress_timezone_warning": false,
    "logo_url": "https://example.com/logo.png"
}
```

## Timezone Handling

The timezone field contains IANA identifiers:

- `America/New_York`
- `America/Chicago`
- `America/Denver`
- `America/Los_Angeles`
- `UTC`
- etc.

Controllers should use this for date/time display:

```php
$timezone = $request['user_context']['timezone'] ?? 'UTC';
$date = new DateTime('now', new DateTimeZone($timezone));
```

## Interface Level

The interface level determines UI complexity and available features:

| Level | Name | Description |
|-------|------|-------------|
| 0 | Guest | Unauthenticated visitor |
| 1 | User | Standard authenticated user |
| 9 | Admin | Full system access |

## Downstream Usage

Controllers access user context for personalized rendering:

```php
public function dashboard($app, array $request): array {
    $context = $request['user_context'] ?? null;
    
    if ($context) {
        $timezone = $context['timezone'] ?? 'UTC';
        $theme = $context['config']['theme'] ?? 'default';
        $interface = $context['interface'] ?? 1;
        
        // Use context for rendering decisions
        return [
            'theme' => $theme,
            'timezone' => $timezone,
            'is_admin' => $interface >= 9,
        ];
    }
    
    // Default for unauthenticated
    return ['theme' => 'default'];
}
```

## Comparison with Identity

| Aspect | Identity (Tier 02) | UserContext (Tier 06) |
|--------|-------------------|----------------------|
| Source | Session data | Database query |
| Performance | Fast (memory) | Slower (I/O) |
| Mutability | Immutable | Mutable (config can change) |
| Use Case | Authentication checks | Feature flags, preferences |
| Always Available | For authenticated users | Only after successful load |

## Error Handling

UserContext does not throw exceptions. If the database query fails or returns no results, the middleware continues without setting `$request['user_context']`.

## Dependencies

- **UserRepository**: `/App/Data/Repositories/UserRepository.php`
- **Identity**: Reads `$request['identity']` from upstream

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 05: InputValidation](05_validation.md)
- [Tier 07: CsrfProtection](07_csrf.md)
- [UI Overview](../ui/OVERVIEW.md)
