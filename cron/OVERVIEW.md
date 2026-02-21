# Cron

DLPR routes cron tasks through the same middleware pipeline as web requests, using a dedicated URI type. This keeps background processing consistent with the rest of the application rather than a separate execution environment.

See [URI_TYPE_CRON.md](../architecture/URI_TYPE_CRON.md) for how cron requests enter and move through the system.

## Core Principles

**Idempotency** — All cron tasks must be safe to run multiple times. If a task runs twice due to overlap or retry, the result should be the same as running it once.

**Locking** — Prevent concurrent execution with a file-based lock. Use atomic file creation (`fopen` with `x` mode) so only one process can acquire the lock. Write a timestamp to the lock file and treat locks older than a configurable timeout as stale.

**Heartbeat** — Write a timestamp to a known file on each successful run. External monitoring checks this file to confirm the cron is alive. Simple, no dependencies.

## Logging

Log meaningful events only — task completed, error encountered. Avoid logging when no work was done. Noise makes real problems harder to find.
