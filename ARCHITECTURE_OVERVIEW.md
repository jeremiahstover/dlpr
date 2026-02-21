# Architecture Overview

System architecture follows the **DLPR** (Data/Logic/Presentation/Routes) pattern with clear separation of concerns and a middleware stack for cross-cutting concerns.

## Core Hubs

Application documentation is organized into focused hubs:

### Architecture Hub
The foundation of the system - pattern definitions, directory structure, and architectural decisions.
- [DLPR Pattern](architecture/DLPR_PATTERN.md) - The Data/Logic/Presentation/Routes architectural pattern
- [Directory Navigator](architecture/DIRECTORY_NAVIGATOR.md) - Quick reference for the filesystem structure
- [Explicit Dependencies](architecture/EXPLICIT_DEPENDENCIES.md) - Dependency injection pattern
- [Route Mapping](architecture/ROUTE_MAPPING.md) - How URLs map to controllers
- [Access Control Architecture](architecture/ACCESS_CONTROL_OVERVIEW.md) - Authorization system design
- [Authorization Patterns](architecture/AUTHORIZATION_CURRENT_STATE.md) - Current authorization implementation

### Feature Hub
Domain-specific documentation for the core features.
- [Studies](features/STUDIES.md) - Bible study management
- [Schedules](features/SCHEDULES.md) - Study scheduling and progression
- [Notifications](features/NOTIFICATIONS.md) - Reminder and delivery system
- [Collections](features/COLLECTIONS.md) - Study grouping and organization

### UI Hub
User interface components and presentation layer details.
- [UI Overview](ui/OVERVIEW.md) - Complete UI system overview
- [Interface Levels](ui/INTERFACE_LEVELS.md) - Dynamic UI complexity
- [Themes](ui/THEMES.md) - Visual theme system
- [Menu System](ui/MENU_SYSTEM.md) - Navigation menu structure
- [Critical Menu Defenses](ui/CRITICAL_MENU_DEFENSES.md) - Menu security safeguards

### Middleware Hub
The request processing stack and cross-cutting concerns.
- [Middleware Overview](middleware/OVERVIEW.md) - Complete middleware stack documentation
- [Access Control Standardization](middleware/ACCESS_CONTROL_STANDARDIZATION.md) - Controller access patterns
- [Tier 01: Config](middleware/01_config.md) - Configuration loading
- [Tier 02: Identity](middleware/02_identity.md) - Authentication and session handling
- [Tier 03: Access](middleware/03_access.md) - Route-level authorization
- [Tier 04: HTTP Override](middleware/04_http_override.md) - HTTP verb normalization
- [Tier 05: Validation](middleware/05_validation.md) - Input validation and sanitization
- [Tier 06: Context](middleware/06_context.md) - User preference loading
- [Tier 07: CSRF](middleware/07_csrf.md) - CSRF token validation

### Cron Hub
Background processing and task automation.
- [Cron Overview](cron/OVERVIEW.md) - Background task lifecycle and monitoring

### Standards Hub
Coding standards, naming conventions, and best practices.
- [Naming Conventions](standards/NAMING_CONVENTIONS.md) - File, class, and variable naming rules
- [Response Patterns](standards/RESPONSE_PATTERNS.md) - Standardized API response shapes
- [Response Builder API](standards/RESPONSE_BUILDER_API.md) - Response helper methods
- [Explicit Dependencies](standards/EXPLICIT_DEPENDENCIES.md) - Dependency injection standards
- [Folder Naming Structure](standards/folder-naming-structure-best-practices.md) - Directory organization guidelines

### Guides Hub
Step-by-step tutorials for common development tasks.
- [Guides Overview](guides/OVERVIEW.md) - Tutorial hub and index
- [Creating Views and Templates](guides/GUIDE_CREATING_VIEWS_AND_TEMPLATES.md) - How to add new UI pages
- [Adding Interface Levels](guides/GUIDE_ADDING_INTERFACE_LEVELS.md) - How to create new UI complexity levels
- [Adding Study Types](guides/GUIDE_ADDING_STUDY_TYPES.md) - How to add new study formats

## Directory Structure

```
/
├── App/
│   ├── Data/                    # DATA LAYER
│   │   ├── Repositories/        # Database repositories (StudyRepository, etc.)
│   │   ├── Clients/             # External API clients (ScheduleApiClient, etc.)
│   │   └── SessionRepository.php
│   ├── Logic/                   # LOGIC LAYER
│   │   ├── Services/            # Business logic (UserService, StudyService)
│   │   └── Middleware/          # Cross-cutting concerns (auth, validation, etc.)
│   ├── Presentation/            # PRESENTATION LAYER
│   │   ├── Presenter.php        # Template rendering orchestrator
│   │   └── Templates/           # HTML templates (Plates format)
│   ├── Routes/
│   │   └── Controllers/         # Route handlers (StudiesController, etc.)
│   ├── App.php                  # Main application orchestrator
│   ├── Bootstrap.php            # Dependency wiring
│   ├── Cron.php                 # Cron job processor
│   ├── Lite.php                 # Lite route handler
│   └── Router.php               # URI routing and classification
├── Cron/                        # Cron job scripts
├── Data/
│   ├── routes.json              # Controller → method mappings
│   ├── aliases.json             # URL shortcuts
│   ├── config.json              # Theme, app configuration
│   ├── menus.json               # Menu definitions
│   ├── paragraphs.json          # Content paragraphs
│   └── init/                    # Database initialization
├── Docs/                        # Documentation organized by purpose
│   ├── current/                 # Active architecture & implementation
│   ├── future/                  # Planned features & roadmap
│   └── past/                    # Completed implementation documentation
├── Files/
│   ├── Database/                # SQLite database files
│   ├── Logs/                    # Application logs
│   └── Cache/                   # Temporary files
├── Lib/                         # Composer dependencies
├── Public/
│   ├── index.php                # Single entry point
│   └── router.php               # PHP dev server router
└── Tests/                       # Unit and integration tests
```

*For a detailed map of the filesystem, see the [Directory Navigator](architecture/DIRECTORY_NAVIGATOR.md).*

## Request Flow

```
Browser → Public/index.php → App.php
    ├─ Load autoloader
    ├─ Instantiate services
    ├─ Parse request
    ├─ Execute middleware stack
    │   ├─ ConfigMiddleware (loads config)
    │   ├─ IdentityMiddleware (sets $request['identity'])
    │   ├─ AccessMiddleware (enforces routes.json rules)
    │   ├─ HttpMethodOverride (normalizes verb)
    │   ├─ InputValidation (sets $request['validated'])
    │   ├─ UserContext (sets $request['user_context'])
    │   └─ CsrfProtection (validates CSRF tokens)
    ├─ ErrorHandler (in App::orchestrate())
    ├─ Router.match() → controller/method
    ├─ Controller → Service → Repository
    └─ Presenter.render() → Template → Response
```

*Detailed breakdown available in the [Middleware Hub](middleware/OVERVIEW.md) and [Life of Request](architecture/LIFE_OF_REQUEST.md).*

## Dependency Management

This project uses an **Explicit Dependency Pattern**. All dependencies required by a class are passed through its constructor.

- **Composition Root**: `App/App.php` is the central point where all services and API clients are initialized.
- **Explicit Injection**: Dependencies are passed down from `App.php` to controllers and services.
- **No Magic**: We avoid static service providers, autowiring, and service locators.

*See [Explicit Dependencies](architecture/EXPLICIT_DEPENDENCIES.md) for implementation details.*

## Interface and Theme Resolution

The system supports multiple interface levels and visual themes, resolved dynamically based on user context.

- **Interface Levels**: [View Levels and Templates](ui/OVERVIEW.md)

## Cron Processing

The cron system manages background tasks in a specific lifecycle:
1. Notifications → 2. Schedules → 3. Studies

*See the [Cron Hub](cron/OVERVIEW.md) for more details.*

---
*For a developer-focused introduction, see [TECHNICAL_OVERVIEW.md](TECHNICAL_OVERVIEW.md).*

---

**Related Docs:**
- [TECHNICAL_OVERVIEW.md](TECHNICAL_OVERVIEW.md) - Developer-focused introduction
- [architecture/LIFE_OF_REQUEST.md](architecture/LIFE_OF_REQUEST.md) - Detailed request lifecycle
- [architecture/DIRECTORY_NAVIGATOR.md](architecture/DIRECTORY_NAVIGATOR.md) - Complete filesystem reference

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
