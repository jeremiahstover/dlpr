# Cron 3-Stage Execution Order

This document provides a deep-dive into the execution order of the ML2 cron system, detailing the three-stage processing pipeline and associated mechanisms for concurrency control and monitoring.

## Overview

The ML2 cron system executes background processing tasks in a specific three-stage order to ensure proper data flow and consistency. Each stage builds upon the results of the previous stage, creating a logical progression from notification delivery to schedule creation to study maintenance.

## Three-Stage Execution Order

The cron execution follows this specific order in `App/Cron.php::run()`:

```php
public function run(): string
{
    // ... locking and initialization
    
    try {
        // Step 1: Run database cleanup to remove orphaned records
        $output .= $this->runDatabaseCleanup();
        
        // Stage 1: Notifications (Due deliveries)
        $output .= $this->notificationProcessor->processNotifications();
        
        // Stage 2: Schedules (Generate next notifications)  
        $output .= $this->scheduleProcessor->processSchedules();
        
        // Stage 3: Studies (Maintenance and cleanup)
        $output .= $this->studyProcessor->processStudies();
        
        $success = true;
    } catch (\Exception $e) {
        // ... error handling
    }
    
    // ... cleanup and logging
}
```

### Stage 1: Notifications (Due Deliveries)

The first stage processes all pending notifications that are due for delivery.

#### Processing Logic

1. **Deactivate Invalid Notifications**: Updates notifications to inactive status where the associated schedule has been deactivated
2. **Fetch Due Notifications**: Retrieves all active notifications where the due date/time is less than or equal to the current time
3. **Process Each Notification**:
   - Mark current notification as inactive (status = 0)
   - Retrieve user contact information (email/phone)
   - Validate delivery method availability (user has email/phone for respective delivery method)
   - Create debug log file in `Files/tmp/`
   - Send notification via appropriate method (email or SMS)
   - Log success/failure status

#### Key Features

- **Delivery Methods**: Supports both email and SMS notifications
- **Contact Validation**: Checks user contact information before attempting delivery
- **Debug Logging**: Creates timestamped log files for each notification attempt
- **Atomic Updates**: Marks notifications as processed before delivery to prevent duplicate sends

#### Error Handling

- Individual notification failures don't halt the entire process
- Failures are logged with specific error details
- Contact information validation prevents delivery attempts to users without required contact methods

### Stage 2: Schedules (Generate Next Notifications)

The second stage creates new notifications for schedules that don't currently have active notifications.

#### Processing Logic

1. **Fetch Schedules Without Active Notifications**: Identifies all active schedules that don't have associated active notifications
2. **Process Each Schedule**:
   - Retrieve associated study and user data
   - Validate schedule conditions (frequency, recurrence rules)
   - Calculate next notification due date/time
   - Build notification content using `NotificationContentBuilder`
   - Build notification subject using `NotificationSubjectBuilder`
   - Create new notification record in database
   - Handle recurrence for recurring schedules

#### Key Features

- **Content Generation**: Dynamically generates notification content based on study data
- **Recurrence Handling**: Properly manages recurring schedules with complex recurrence patterns
- **Time Zone Awareness**: Respects user time zones for notification timing
- **API Integration**: Fetches external data as needed for content generation

#### Dependencies

- Relies on successful completion of Stage 1 to avoid conflicts with existing notifications
- Requires access to external APIs for dynamic content generation

### Stage 3: Studies (Maintenance and Cleanup)

The third stage performs maintenance operations on studies, ensuring they have proper schedules and cleaning up as needed.

#### Processing Logic

1. **Fetch Active Studies Without Schedules**: Identifies all active studies that don't have associated active schedules
2. **Process Each Study**:
   - Retrieve study details and user preferences
   - Generate initial schedule based on study configuration
   - Handle special cases (collections, verses, topical content)
   - Create schedule record in database
   - Handle recurrence patterns and frequency settings

#### Key Features

- **Initial Schedule Creation**: Automatically creates schedules for new studies
- **Content Integration**: Supports various content types (collections, verses, topical)
- **Frequency Management**: Handles different study frequencies and patterns
- **API Coordination**: Integrates with external APIs for content retrieval

#### Dependencies

- Relies on successful completion of Stages 1 and 2 to maintain data consistency
- Requires access to external APIs for content scheduling

## Locking Mechanism

To prevent concurrent execution and race conditions, the cron system implements a file-based locking mechanism.

### Implementation

```php
class CronLocking
{
    public function acquireLock()
    {
        $fp = fopen($this->lockFilePath, 'x'); // Atomic: fails if exists
        if ($fp === false) {
            // Lock exists, check age
            if (file_exists($this->lockFilePath) && (time() - filemtime($this->lockFilePath)) > $this->lockTimeout) {
                unlink($this->lockFilePath); // Stale lock, remove
                $fp = fopen($this->lockFilePath, 'x');
            }
        }
        if ($fp !== false) {
            file_put_contents($this->lockFilePath, time()); // Write timestamp
            return $fp;
        }
        return null; // Locked by active process
    }
}
```

### Key Features

- **Atomic Lock Acquisition**: Uses `fopen()` with 'x' mode to atomically create the lock file
- **Stale Lock Handling**: Automatically removes locks older than the timeout period (default 1 hour)
- **Process Identification**: Stores process ID in lock file for identification
- **Graceful Failure**: Returns null when lock cannot be acquired, allowing caller to handle appropriately

### Error Handling

- Prevents multiple concurrent cron executions
- Automatically cleans up stale locks
- Provides clear error messages when locked

## Heartbeat System

The cron system maintains a heartbeat file for monitoring purposes.

### Implementation

```php
// Touch the heartbeat file to indicate cron is running
$this->logger->touchHeartbeatFile();
```

### Purpose

- **Monitoring**: Allows external systems to monitor cron health
- **Alerting**: Enables alerts when cron stops running
- **Status Tracking**: Provides timestamp of last successful execution

## Database Cleanup

Before the three-stage processing begins, the system performs database cleanup to remove orphaned records.

### Cleanup Operations

1. **Orphaned Studies**: Deletes studies without associated users
2. **Orphaned Schedules**: Deletes schedules without associated studies
3. **Orphaned Notifications**: Deletes notifications without associated schedules

### Importance

- **Data Integrity**: Maintains referential integrity in the database
- **Performance**: Removes unnecessary records that could impact query performance
- **Resource Management**: Frees storage space from obsolete data

## Error Handling and Recovery

The cron system implements comprehensive error handling to ensure robust operation.

### Exception Handling

```php
try {
    // ... processing stages
    $success = true;
} catch (\Exception $e) {
    $errorMsg = "FATAL ERROR: " . $e->getMessage();
    $output .= $errorMsg . "\n";
    error_log("Cron Fatal Error: " . $e->getMessage());
    
    // Log the error to file
    $this->logger->logToFile("ERROR: " . $e->getMessage());
    if (method_exists($e, 'getTraceAsString')) {
        $this->logger->logToFile("ERROR TRACE: " . $e->getTraceAsString());
    }
}
```

### Recovery Strategies

- **Individual Failure Isolation**: Errors in one stage don't necessarily halt subsequent stages
- **Detailed Logging**: Comprehensive error logging for debugging and monitoring
- **Graceful Degradation**: System continues operating even when non-critical components fail

## Monitoring and Logging

The cron system provides extensive logging for monitoring and debugging purposes.

### Log Categories

1. **Informational**: Normal operation logging
2. **Warnings**: Non-critical issues that should be addressed
3. **Errors**: Critical failures requiring immediate attention

### Log Destinations

- **File Logs**: Detailed logs stored in configured log directory
- **Error Logs**: PHP error log for critical system errors
- **Console Output**: Basic status information for manual execution

## Performance Considerations

### Resource Management

- **Database Connections**: Efficiently managed throughout execution
- **Memory Usage**: Objects are properly released after use
- **File Operations**: Minimized and batched where possible

### Execution Time

- **Parallel Processing**: Where safe, operations are designed to minimize total execution time
- **Timeout Handling**: Long-running operations respect system timeouts
- **Progress Tracking**: Large batches provide periodic status updates

## Security Considerations

### Access Control

- **Process Isolation**: Cron execution is isolated from web processes
- **File Permissions**: Proper file permissions on log and lock files
- **Data Validation**: Input validation on all processed data

### Data Protection

- **Sensitive Information**: Careful handling of user contact information
- **API Credentials**: Secure storage and usage of external API credentials
- **Audit Trail**: Comprehensive logging for security auditing