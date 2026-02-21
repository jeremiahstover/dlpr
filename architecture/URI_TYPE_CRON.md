# URI Type: Cron Routes

## Classification Logic

```php
// Cron job endpoint
if ($uri === '/cron') {
    return ['type' => 'cron', 'resolved_uri' => $uri];
}
```

## Processing Pipeline

Cron routes are handled by `App/Cron.php` which implements a 3-stage execution order:

```php
// Cron.php::run() - Execution Order
public function run(): string {
    // Stage 1: Notifications (Due deliveries)
    $this->notificationProcessor->processNotifications();
    
    // Stage 2: Schedules (Generate next notifications)  
    $this->scheduleProcessor->processSchedules();
    
    // Stage 3: Studies (Maintenance and cleanup)
    $this->studyProcessor->processStudies();
}
```

## Three-Stage Execution Order

1. **Notifications**: Process due deliveries and notifications
2. **Schedules**: Generate next notifications and schedule future tasks
3. **Studies**: Perform maintenance and cleanup operations on studies

## Locking Mechanism

To prevent concurrent execution, the cron system implements a file-based locking mechanism:

```php
// Prevents race conditions during overlapping executions
$lockFile = '/Files/tmp/cron.lock';
if (file_exists($lockFile)) {
    $pid = file_get_contents($lockFile);
    if (file_exists("/proc/$pid")) {
        throw new Exception('Cron already running');
    }
}
// Write current PID and proceed
```

## Heartbeat System

Execution status is tracked in `cron-heartbeat.txt` for monitoring purposes.

## Error Handling

Exceptions during cron execution are logged but don't halt the entire process, allowing subsequent stages to continue.

---

**See Also:**
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - Overview of all URI types
- [EXECUTION_ORDER.md](../cron/EXECUTION_ORDER.md) - Detailed cron execution order

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete