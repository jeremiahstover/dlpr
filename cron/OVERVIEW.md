# Cron Lifecycle and Heartbeat

The cron system manages background tasks such as notification delivery, schedule updates, and study maintenance.

## Execution Cycle

The cron runs periodically (usually every minute) and processes tasks in the following order:

1.  **Notifications**: Checks for pending notifications that are due for delivery.
2.  **Schedules**: Processes schedule progressions and generates the next notifications.
3.  **Studies**: Performs maintenance on studies and cleans up completed or inactive records.

## Monitoring

### Heartbeat File
The cron system writes its current status to a heartbeat file located at:
`/Files/Logs/cron-heartbeat.txt`

This file contains the timestamp of the last successful cron execution. Monitoring systems can check this file to ensure the cron is running as expected.

### Logging
Detailed cron logs are written to `/Files/Logs/cron.log`. To reduce noise, cron only logs meaningful events (e.g., notification sent, error encountered) and avoids logging when no work is performed.

## Best Practices
-   **Idempotency**: All cron tasks must be idempotent (safe to run multiple times).
-   **Locking**: The cron system ensures only one instance is running at a time to prevent race conditions.
-   **Timeout**: Long-running tasks should be broken into batches to avoid exceeding execution limits.

---
**Related Docs:**
- [ARCHITECTURE_OVERVIEW.md](../ARCHITECTURE_OVERVIEW.md)
- [URI_TYPE_CRON.md](../architecture/URI_TYPE_CRON.md) - Cron route processing
- [EXECUTION_ORDER.md](EXECUTION_ORDER.md) - Detailed execution order
- [NOTIFICATIONS.md](../features/NOTIFICATIONS.md)
- [SCHEDULES.md](../features/SCHEDULES.md)

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete
