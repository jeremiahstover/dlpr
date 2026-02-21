# Folder Naming and Structure Best Practices

This guide documents the mandatory folder structure and naming conventions for the Application repository.

## Documentation Hierarchy

The documentation is organized into three top-level categories based on the status of the information:

- **`Docs/current/`**: The "Standards Hub". Contains only high-signal architecture reference documents and current system invariants. This is the source of truth for how the system works *now*.
- **`Docs/future/`**: Contains planned features, roadmaps, and RFCs for upcoming work.
- **`Docs/past/`**: The archive for implementation logs, fix summaries, feature histories, and historical narratives.

## STRICT CONSTRAINT: No Implementation Narratives in `current/`

The `Docs/current/` directory must remain a clean reference for architecture and standards. 

- **Do NOT** include implementation logs or fix summaries in `current/`.
- **Do NOT** include historical narratives about how a feature was built.
- **Move** all task-specific summaries and "lessons learned" from individual tickets to `Docs/past/` once the work is complete.

The `current/` directory is for **invariants**: the rules and patterns that always apply to the codebase.

## Self-Documenting Layered Structure (DLPR)

The `App/` directory is organized into four distinct layers:

1.  **Data (`App/Data/`)**: Responsible for data access.
    - `Clients/`: Wrappers for external APIs.
    - `Repositories/`: Database access and mapping to domain arrays.
2.  **Logic (`App/Logic/`)**: Responsible for business rules and request processing.
    - `Services/`: Domain-specific business logic.
    - `Middleware/`: Cross-cutting concerns (Auth, Validation, CSRF).
    - `Helpers/`: Reusable, stateless utility functions.
3.  **Presentation (`App/Presentation/`)**: Responsible for view templates.
    - Organized by domain (e.g., `Studies/`, `Admin/`, `User/`).
4.  **Routes (`App/Routes/`)**: Responsible for entry points.
    - `Controllers/`: Orchestrate the flow between logic and presentation.

## Naming Principles

- **Folders**: Use PascalCase for code directories (e.g., `Repositories/`) and simple nouns. No abbreviations (use `Repositories/` not `Repos/`).
- **Files**: Use PascalCase for PHP classes. The filename must match the class name exactly.
- **Consistency**: Suffixes should be used consistently to identify the role of a class:
    - `*Controller.php`
    - `*Service.php`
    - `*Repository.php`
    - `*Client.php`
    - `*Middleware.php`
    - `*Helper.php`
