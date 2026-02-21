# Architecture Deep Dive

This document provides a comprehensive technical deep dive into the Application architecture, consolidating insights from individual architecture documents to present a holistic view of the system design.

## Table of Contents

1. [System Entry Points](#system-entry-points)
2. [Request Processing Pipeline](#request-processing-pipeline)
3. [URI Type Classification and Routing](#uri-type-classification-and-routing)
4. [Autoloading System](#autoloading-system)
5. [Route Aliasing Mechanism](#route-aliasing-mechanism)
6. [API Content Negotiation](#api-content-negotiation)
7. [Middleware Stack](#middleware-stack)
8. [DLPR Architecture Pattern](#dlpr-architecture-pattern)
9. [Explicit Dependencies](#explicit-dependencies)
10. [Cron System Execution Order](#cron-system-execution-order)
11. [Security Mechanisms](#security-mechanisms)
12. [Directory Structure](#directory-structure)

## System Entry Points

The Application application has a single entry point at `Public/index.php` that handles all incoming requests. This entry point is responsible for:

- Loading the autoloader
- Instantiating the main application class
- Processing static file requests directly
- Delegating dynamic requests to the application orchestrator

For detailed information on entry points, see [ENTRY_POINTS.md](ENTRY_POINTS.md).

## Request Processing Pipeline

The complete life of a request in Application follows this path:

1. Entry point processing in `Public/index.php`
2. Application orchestration in `App/App.php`
3. Request parsing and normalization
4. Middleware stack execution
5. Route matching and controller identification
6. Controller execution with service layer interaction
7. Output rendering through the presentation layer
8. Response sending back to the client
9. Exception handling and error response generation

For a detailed breakdown, see [LIFE_OF_REQUEST.md](LIFE_OF_REQUEST.md).

## URI Type Classification and Routing

Application classifies incoming requests into five distinct URI types:

1. **Static** - Direct file serving
2. **Lite** - Simple page requests
3. **Cron** - Background task endpoints
4. **Heavy** - Complex application requests
5. **API** - Programmatic interface requests

Each URI type follows a specific processing pipeline with unique security and performance characteristics. The classification logic determines how requests are handled throughout their lifecycle.

For detailed information, see [URI_TYPES.md](URI_TYPES.md).

## Autoloading System

Application implements a custom autoloader that follows a two-part logic:

1. Application code autoloading based on namespace mapping
2. External library autoloading for third-party dependencies

The autoloader is designed to be PSR-4 compliant while providing optimized performance through selective class loading and caching strategies.

For implementation details, see [AUTOLOADER.md](AUTOLOADER.md).

## Route Aliasing Mechanism

The route aliasing system provides URL shortcuts defined in `Data/aliases.json`. Key features include:

- Pattern-based alias definitions
- Precedence rules for conflict resolution
- Legacy controller mappings for backward compatibility
- API endpoint aliasing support

For complete details, see [ROUTE_ALIASES.md](ROUTE_ALIASES.md).

## API Content Negotiation

Application supports API content negotiation through:

- URI-based detection (`/api/` prefix)
- HTTP header-based negotiation (`Accept: application/json`)
- AJAX detection (`X-Requested-With: XMLHttpRequest`)
- Format-specific request/response handling
- Authentication differences between API and web requests

For detailed information, see [API_NEGOTIATION_OVERVIEW.md](API_NEGOTIATION_OVERVIEW.md) and [API_NEGOTIATION_GUIDE.md](API_NEGOTIATION_GUIDE.md).

## Middleware Stack

The middleware stack executes in a specific order to process cross-cutting concerns:

1. ConfigMiddleware - Loads application configuration
2. IdentityMiddleware - Sets request identity context
3. AccessMiddleware - Enforces route access controls
4. HttpMethodOverrideMiddleware - Normalizes HTTP verbs
5. InputValidationMiddleware - Validates and sanitizes inputs
6. ContextMiddleware - Establishes request context
7. CsrfProtection - Validates CSRF tokens

Each middleware has a focused responsibility and contributes to the overall request processing pipeline.

## DLPR Architecture Pattern

Application is organized around the DLPR pattern:

- **Data** (`/App/Data/`) - Persistence and integration layer
- **Logic** (`/App/Logic/`) - Business rules and request processing
- **Presentation** (`/App/Presentation/`) - HTML rendering and template selection
- **Routes** (`/App/Routes/`) - Request routing and controller dispatch

This separation ensures clear boundaries between concerns and facilitates maintainability.

For detailed information, see [DLPR_PATTERN.md](DLPR_PATTERN.md).

## Explicit Dependencies

All classes in Application follow the explicit dependency pattern, where dependencies are injected through constructors rather than accessed through static methods or service locators. This approach:

- Improves testability
- Makes dependencies explicit
- Reduces coupling between components
- Enables easier refactoring

The composition root in `App/App.php` manages all dependency wiring.

For implementation details, see [EXPLICIT_DEPENDENCIES.md](EXPLICIT_DEPENDENCIES.md).

## Cron System Execution Order

The cron system executes background tasks in a specific order:

1. Notifications processing
2. Schedule processing
3. Study processing

This order ensures data consistency and proper timing of dependent operations. The system includes locking mechanisms and heartbeat monitoring.

For detailed information, see [EXECUTION_ORDER.md](../cron/EXECUTION_ORDER.md).

## Security Mechanisms

Application implements several security mechanisms to protect against common vulnerabilities:

### Path Traversal Prevention

Multiple layers of defense prevent directory traversal attacks:

- Input sanitization and validation
- Realpath-based path resolution
- Whitelist-based file access controls
- Component-specific protection mechanisms

For detailed information, see [PATH_TRAVERSAL_OVERVIEW.md](PATH_TRAVERSAL_OVERVIEW.md) and [PATH_TRAVERSAL_GUIDE.md](PATH_TRAVERSAL_GUIDE.md).

### Static File Serving Security

Security measures for static file serving include:

- Directory traversal prevention
- MIME type validation
- File extension whitelisting
- Development server security considerations

For detailed information, see [STATIC_FILE_SERVING_SECURITY.md](STATIC_FILE_SERVING_SECURITY.md).

## Directory Structure

The Application directory structure reflects the DLPR architecture pattern:

```
/
├── App/
│   ├── Data/                    # DATA LAYER
│   │   ├── Repositories/        # Database repositories
│   │   ├── Clients/             # External API clients
│   │   └── SessionRepository.php
│   ├── Logic/                   # LOGIC LAYER
│   │   ├── Services/            # Business logic
│   │   └── Middleware/          # Cross-cutting concerns
│   ├── Presentation/            # PRESENTATION LAYER
│   │   ├── Presenter.php        # Template rendering orchestrator
│   │   └── Templates/           # HTML templates
│   ├── Routes/
│   │   └── Controllers/         # Route handlers
│   ├── App.php                  # Main application orchestrator
│   └── Bootstrap.php            # Dependency wiring
├── Data/
│   ├── routes.json              # Controller → method mappings
│   ├── aliases.json             # URL shortcuts
│   └── config.json              # Theme, app configuration
├── Docs/                        # Documentation organized by purpose
│   ├── current/                 # Active architecture & implementation
│   ├── future/                  # Planned features & roadmap
│   └── past/                    # Completed implementation documentation
├── Public/
│   ├── index.php                # Single entry point
│   └── router.php               # PHP dev server router
├── Files/
│   ├── Database/                # SQLite database files
│   └── Logs/                    # Application logs
└── Tests/                       # Unit and integration tests
```

For a detailed map of the filesystem, see the [Directory Navigator](DIRECTORY_NAVIGATOR.md).
---

**Document Version:** 1.1
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
