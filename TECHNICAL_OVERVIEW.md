# Technical Overview (ML2)

This is the developer entry point for the Memorize Live ML2 codebase: **how it’s organized**, **where to find things**, and **how a request moves through the system**.

## Documentation Hubs

To navigate the system's documentation, use the following hubs:

1. [**Architecture Hub**](architecture/ARCHITECTURE_DEEP_DIVE.md) - Comprehensive deep dive into ML2 architecture
2. [**UI & Presentation Hub**](ui/OVERVIEW.md) - Interface levels, themes, and menu defenses
3. [**Middleware Hub**](middleware/OVERVIEW.md) - The request processing stack and access control
4. [**Feature Hub**](features/STUDIES.md) - Domain-specific documentation for Studies, Schedules, Notifications, and Collections
5. [**Cron Hub**](cron/OVERVIEW.md) - Background tasks and heartbeat monitoring.
6. [**Standards Hub**](standards/NAMING_CONVENTIONS.md) - Naming conventions and API response shapes.
7. [**Guides Hub**](guides/OVERVIEW.md) - Step-by-step tutorials for common development tasks.

---

## 1. Core Architecture

ML2 is organized around **DLPR**: **Data → Logic → Presentation → Routes**. Each layer has a narrow responsibility.

- **Data (`/App/Data/`)**: Owns persistence and integration details. [Learn more](architecture/DLPR_PATTERN.md#1-data-layer-appdata).
- **Logic (`/App/Logic/`)**: Owns business rules and cross-cutting request processing. [Learn more](architecture/DLPR_PATTERN.md#2-logic-layer-applogic).
- **Presentation (`/App/Presentation/`)**: Owns HTML rendering and template selection. [Learn more](architecture/DLPR_PATTERN.md#3-presentation-layer-apppresentation).
- **Routes (`/App/Routes/`)**: Owns request routing and controller selection. [Learn more](architecture/DLPR_PATTERN.md#4-routes-layer-approutes).

## 2. Request Lifecycle

- **Middleware First**: All requests pass through a stack of middleware before reaching a controller. [See Middleware Order](middleware/OVERVIEW.md#execution-order).
- **Explicit Dependencies**: Dependencies are injected via constructors, with wiring managed in `/App/App.php`. [See Dependency Pattern](architecture/EXPLICIT_DEPENDENCIES.md).
- **Centralized Error Handling**: Exceptions are caught and processed by a dedicated ErrorHandler.

## 3. Data Layer

- **Repositories**: Responsible for SQL queries and row mapping. Located in `/App/Data/Repositories/`.
- **API Clients**: External integrations (Verse API, Mailgun, etc.) live in `/App/Data/Clients/`.

## 4. UI System

- **Pico CSS**: Base styling with semantic HTML.
- **Interface Levels**: Dynamic UI complexity based on user level. [Learn more](ui/OVERVIEW.md#1-interface-levels).
- **Theme System**: Support for multiple visual themes. [Learn more](ui/OVERVIEW.md#2-theme-system).

---

## 5. Development Standards

- **Naming Conventions**: Follow the standards defined in the [Standards Hub](standards/NAMING_CONVENTIONS.md).
- **Response Shapes**: Controllers must return standardized arrays via `ResponseBuilder`. [See Response Patterns](standards/RESPONSE_PATTERNS.md).
- **Testing**: All new features must include tests in the `/Tests/` directory.

---
*For a comprehensive architectural deep dive, see [ARCHITECTURE_DEEP_DIVE.md](architecture/ARCHITECTURE_DEEP_DIVE.md).*

---

**Related Docs:**
- [ARCHITECTURE_OVERVIEW.md](ARCHITECTURE_OVERVIEW.md) - Master hub entry point
- [architecture/ARCHITECTURE_DEEP_DIVE.md](architecture/ARCHITECTURE_DEEP_DIVE.md) - Comprehensive architectural deep dive
- [architecture/LIFE_OF_REQUEST.md](architecture/LIFE_OF_REQUEST.md) - Detailed request lifecycle

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
