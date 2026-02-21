# Tier 02: IdentityMiddleware

**Purpose**: Determine the authenticated user's identity from session data and enforce authentication requirements for protected routes.

## Enrichment

| Key Added | Type | Description |
|-----------|------|-------------|
| `$request['identity']` | `Identity\|null` | Immutable value object representing the authenticated user, or `null` if unauthenticated |

## The Identity Value Object

`Identity` is a readonly PHP 8.2+ class that enforces type safety and immutability:

```php
readonly class Identity {
    public function __construct(
        public int $id,
        public string $email,
        public ?string $name = null,
        public int $interface = 1,
        public ?string $timezone = 'UTC',
        public ?string $theme = null,
        public ?string $role = 'guest'
    );
}
```

### Identity Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `int` | Unique user ID (must be > 0) |
| `email` | `string` | Validated email address |
| `name` | `?string` | Display name (defaults to email) |
| `interface` | `int` | Interface level (0-9, default 1) |
| `timezone` | `?string` | Valid timezone identifier |
| `theme` | `?string` | Preferred theme identifier |
| `role` | `?string` | One of: `guest`, `user`, `admin` |

## Responsibilities

1. **Session Validation**: Retrieves authentication data from `SessionRepository`
2. **Identity Creation**: Converts session data to immutable `Identity` object
3. **Role Determination**: Calculates role based on `interface_map` and `admins` config
4. **Authentication Enforcement**: Throws 401 for protected routes when unauthenticated
5. **Special Route Handling**: Manages login/logout flows without blocking them

## Session Establishment Routes

These routes bypass normal authentication to allow login:

```php
private const SESSION_ESTABLISHMENT_ROUTES = [
    '/user/login',
    '/validate-login',
];
```

## Session Clearance Routes

These routes clear the session (logout):

```php
private const SESSION_CLEARANCE_ROUTES = [
    '/user/logout',
];
```

## Role Determination Logic

Role is determined in the following priority:

1. **Admin via Interface Map**: If `interface_map[interface] === 'admin'`
2. **Admin via Email**: If email exists in `config['admins']`
3. **Default**: `user` (for authenticated), `guest` (for unauthenticated)

```php
private function determineRole(array $identityData, array $request): string {
    // Check interface map for admin role
    $interfaceValue = (int)($identityData['interface'] ?? 0);
    $interfaceMap = $request['config']['interface_map'] ?? [];
    
    if ($interfaceMap[(string)$interfaceValue] === 'admin') {
        return 'admin';
    }
    
    // Check admin email list
    $email = (string)($identityData['email'] ?? '');
    $adminsConfig = $request['config']['admins'] ?? [];
    if ($this->isAdminEmail($email, $adminsConfig)) {
        return 'admin';
    }
    
    return 'user';
}
```

## Authentication Flow

```
Request arrives
       │
       ▼
┌─────────────────┐
│ Is session      │─No──► Check Authorization header (API stub)
│ established?    │
└─────────────────┘
       │ Yes
       ▼
┌─────────────────┐
│ Create Identity │
│ from session    │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ Is this a       │─Yes──► Handle login/logout special cases
│ special route?  │
└─────────────────┘
       │ No
       ▼
┌─────────────────┐
│ Check if public │─Yes──► Continue with identity set
│ route (routes.  │
│ json)           │
└─────────────────┘
       │ No (protected)
       ▼
┌─────────────────┐
│ Identity null?  │─Yes──► Throw 401 Unauthorized
└─────────────────┘
       │ No
       ▼
   Continue
```

## Error Handling

| Exception | Code | Condition |
|-----------|------|-----------|
| `RuntimeException` | 401 | Protected route accessed without valid identity |

## Downstream Usage

Middleware that depends on identity:

| Middleware | Identity Usage |
|-----------|---------------|
| AccessMiddleware | Reads `$request['identity']` to enforce access rules |
| UserContext | Uses `$request['identity']->id` to load user preferences |

Controllers access identity via:

```php
$identity = $request['identity'];

if ($identity !== null) {
    $userId = $identity->id;
    $isAdmin = $identity->role === 'admin';
    $timezone = $identity->timezone;
}
```

## API Authentication Stub

The middleware includes a stub for future API authentication via Authorization headers:

```php
if (empty($identityData) && isset($request['headers']['Authorization'])) {
    $identityData = $this->authenticateViaHeader($request['headers']['Authorization']);
}
```

Currently, this returns `null` and falls back to session-based authentication.

## Dependencies

- **SessionRepository**: Retrieves session auth data
- **RoutesRepository**: Determines if route is public from `routes.json`
- **UserService**: Optional, for admin email checking
- **Identity**: `/App/Logic/Helpers/Identity.php`

## Related Documentation

- [Middleware Overview](OVERVIEW.md)
- [Tier 01: ConfigMiddleware](01_config.md)
- [Tier 03: AccessMiddleware](03_access.md)
