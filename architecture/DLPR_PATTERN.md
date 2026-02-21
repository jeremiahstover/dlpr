# DLPR Pattern

The MemorizeLive ML2 project follows the **DLPR** architectural pattern, which stands for **D**ata, **L**ogic, **P**resentation, and **R**outes. This pattern ensures a clean separation of concerns and a maintainable codebase.

## The Four Layers

### 1. Data Layer (`App/Data/`)
Responsible for all data persistence and external API communication.
- **Repositories**: Handle database queries (e.g., `StudyRepository`, `ScheduleRepository`). They should only contain SQL logic and return data objects or arrays.
- **Clients**: Wrappers for external APIs (e.g., `MailgunClient`, `SignalWireClient`).

### 2. Logic Layer (`App/Logic/`)
Contains the business logic and cross-cutting concerns.
- **Services**: Orchestrate business rules and coordinate between repositories and clients.
- **Middleware**: Handles cross-cutting concerns like authentication, authorization, validation, and CSRF protection before the request reaches the controller.
- **Helpers**: Stateless, reusable utility functions.

### 3. Presentation Layer (`App/Presentation/`)
Handles the rendering of the user interface.
- **Templates**: PHP files using the Plates template engine.
- **Presenter**: Orchestrates template rendering, including theme and interface level resolution.

### 4. Routes Layer (`App/Routes/`)
The entry point for application logic.
- **Controllers**: Receive pre-processed requests from the middleware, call the appropriate services, and return data to be rendered by the presentation layer.

## Request Flow
1. **Public/index.php**: Single entry point.
2. **App.php**: Orchestrates the request.
3. **Middleware Stack**: Processes the request (Auth, Validation, etc.).
4. **Router**: Matches the URI to a Controller and Method.
5. **Controller**: Calls Services.
6. **Service**: Uses Repositories/Clients to get/set data.
7. **Presenter**: Renders the final response using Templates.

## Separation of Concerns
- **Controllers** should NEVER contain business logic, database queries, or authorization checks.
- **Services** should NEVER handle HTTP requests or responses directly.
- **Repositories** should NEVER contain business rules.
- **Templates** should NEVER perform data fetching or complex logic.

---
**Related Docs:**
- [ARCHITECTURE_OVERVIEW.md](../ARCHITECTURE_OVERVIEW.md)
- [DIRECTORY_NAVIGATOR.md](DIRECTORY_NAVIGATOR.md)
- [ENTRY_POINTS.md](ENTRY_POINTS.md)
- [ROUTE_MAPPING.md](ROUTE_MAPPING.md)

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
