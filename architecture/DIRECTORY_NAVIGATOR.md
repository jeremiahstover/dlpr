# Directory Navigator

A quick-reference guide to the Application directory structure.

## Core Application (`/App/`)

| Directory | Layer | Purpose |
|-----------|-------|---------|
| `Cron/` | Cron | Scheduled task handlers |
| `Data/Clients/` | Data | External API integrations |
| `Data/Repositories/` | Data | Database access logic (SQL) |
| `Logic/Helpers/` | Logic | Stateless utility functions |
| `Logic/Middleware/` | Logic | Request preprocessing (Auth, CSRF, etc.) |
| `Logic/Services/` | Logic | Business logic and orchestration |
| `Presentation/Templates/` | Presentation | UI templates (organized by domain) |
| `Routes/Controllers/` | Routes | Request handlers |
| `Routes/Router.php` | Routes | URI routing and route type detection |
| `App.php` | Core | Main application container and middleware orchestration |
| `Cron.php` | Core | Cron job runner |
| `Lite.php` | Core | Lightweight request handler for lite routes |

## Configuration and Metadata (`/Data/`)

- `routes.json`: Primary mapping of URIs to Controller/Method and access levels.
- `config.json`: Application-wide settings (theme, base URL, etc.).
- `aliases.json`: URL shorteners and redirects.
- `menus.json`: Navigation menu definitions.
- `paragraphs.json`: Content paragraphs for static pages.
- `init/`: Database initialization scripts.

## Runtime Files (`/Files/`)

- `Database/`: SQLite database files.
- `Logs/`: Application logs, including `cron-heartbeat.txt`.
- `Cache/`: Temporary application cache.

## Public Entry Point (`/Public/`)

- `index.php`: The main entry point for all web requests.
- `Assets/`: Static files (CSS, JS, Images).

## Dependencies (`/Lib/`)

- `autoload.php`: Composer autoloader for external dependencies.

## Tests (`/Tests/`)

- Flat structure containing unit and integration tests.
- `run-tests.php`: Test runner script.

## Documentation (`/Docs/`)

- `current/`: Active architecture and technical guides.
- `future/`: Planned features and roadmap.
- `past/`: Completed project implementation summaries.

---

**Related Docs:**
- [ARCHITECTURE_OVERVIEW.md](../ARCHITECTURE_OVERVIEW.md)
- [ENTRY_POINTS.md](ENTRY_POINTS.md)
- [AUTOLOADER.md](AUTOLOADER.md)

**Document Version:** 1.1  
**Last Updated:** 2025-02-08  
**Status:** Accurate and Complete