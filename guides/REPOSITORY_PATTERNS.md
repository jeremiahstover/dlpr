# Repository Patterns

Repositories encapsulate data access logic, providing a clean API for querying and persisting domain objects. They isolate the application from database details.

## Overview

Repositories in Application follow these principles:
- **Single Table**: Each repository handles one database table
- **PDO Access**: Direct PDO usage for queries
- **Row Mapping**: Convert database rows to domain arrays
- **No Business Logic**: Repositories query data, services process it

## Architecture

```
App/Data/Repositories/
├── UserRepository.php
├── StudiesRepository.php
├── ScheduleRepository.php
├── CollectionRepository.php
├── NotificationRepository.php
├── ConfigRepository.php
├── SessionRepository.php
└── RoutesRepository.php
```

## Repository Structure

### Basic Repository Template

```php
<?php
declare(strict_types=1);

namespace Application\App\Data\Repositories;

class MyRepository {
    private \PDO $pdo;
    
    public function __construct(\PDO $pdo) {
        $this->pdo = $pdo;
    }
    
    /**
     * Map database row to domain array
     */
    private function mapToDomain(?array $row): ?array {
        if (!$row) {
            return null;
        }
        
        // Rename columns if needed
        if (array_key_exists('created', $row)) {
            $row['created_at'] = $row['created'];
            unset($row['created']);
        }
        
        return $row;
    }
    
    /**
     * Find by ID
     */
    public function findById(int $id): ?array {
        $stmt = $this->pdo->prepare('SELECT * FROM my_table WHERE id = ?');
        $stmt->execute([$id]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $this->mapToDomain($row);
    }
    
    /**
     * Get all with pagination
     */
    public function getAll(int $limit, int $offset): array {
        $stmt = $this->pdo->prepare('
            SELECT * FROM my_table 
            ORDER BY created DESC 
            LIMIT ? OFFSET ?
        ');
        $stmt->bindValue(1, $limit, \PDO::PARAM_INT);
        $stmt->bindValue(2, $offset, \PDO::PARAM_INT);
        $stmt->execute();
        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
        return array_map([$this, 'mapToDomain'], $rows);
    }
    
    /**
     * Count all records
     */
    public function countAll(): int {
        $stmt = $this->pdo->query('SELECT COUNT(*) FROM my_table');
        return (int)$stmt->fetchColumn();
    }
    
    /**
     * Save (insert or update)
     */
    public function save(array $data): int {
        if (!empty($data['id'])) {
            return $this->update($data);
        }
        return $this->insert($data);
    }
    
    /**
     * Insert new record
     */
    private function insert(array $data): int {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));
        
        $stmt = $this->pdo->prepare(
            "INSERT INTO my_table ({$columns}) VALUES ({$placeholders})"
        );
        $stmt->execute(array_values($data));
        
        return (int)$this->pdo->lastInsertId();
    }
    
    /**
     * Update existing record
     */
    private function update(array $data): int {
        $id = $data['id'];
        unset($data['id']);
        
        $sets = [];
        $values = [];
        foreach ($data as $column => $value) {
            $sets[] = "{$column} = ?";
            $values[] = $value;
        }
        $values[] = $id;
        
        $stmt = $this->pdo->prepare(
            "UPDATE my_table SET " . implode(', ', $sets) . " WHERE id = ?"
        );
        $stmt->execute($values);
        
        return $id;
    }
    
    /**
     * Delete by ID
     */
    public function delete(int $id): bool {
        $stmt = $this->pdo->prepare('DELETE FROM my_table WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->rowCount() > 0;
    }
}
```

## Common Patterns

### Find by Foreign Key

```php
public function getByUserId(int $userId): array {
    $stmt = $this->pdo->prepare('
        SELECT * FROM studies 
        WHERE user_id = ? 
        ORDER BY created DESC
    ');
    $stmt->execute([$userId]);
    $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
    return array_map([$this, 'mapToDomain'], $rows);
}
```

### Join Query

```php
public function getWithUser(int $id): ?array {
    $stmt = $this->pdo->prepare('
        SELECT s.*, u.email as user_email 
        FROM studies s 
        LEFT JOIN users u ON s.user_id = u.id 
        WHERE s.id = ?
    ');
    $stmt->execute([$id]);
    $row = $stmt->fetch(\PDO::FETCH_ASSOC);
    return $this->mapToDomain($row);
}
```

### Complex Query with Multiple Joins

```php
public function getStudiesWithNextDue(int $userId, int $limit, int $offset): array {
    $stmt = $this->pdo->prepare('
        SELECT 
            s.*,
            COUNT(DISTINCT sch.id) as schedule_count,
            COUNT(DISTINCT n.id) as notification_count,
            MIN(CASE 
                WHEN n.sent = 0 AND n.send_at > datetime("now") 
                THEN n.send_at 
                ELSE NULL 
            END) as next_due
        FROM studies s
        LEFT JOIN schedules sch ON s.id = sch.study_id
        LEFT JOIN notifications n ON sch.id = n.schedule_id
        WHERE s.user_id = ?
        GROUP BY s.id
        ORDER BY s.created DESC
        LIMIT ? OFFSET ?
    ');
    $stmt->execute([$userId, $limit, $offset]);
    $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
    return array_map([$this, 'mapToDomain'], $rows);
}
```

### Search with LIKE

```php
public function searchByTitle(string $query, int $userId): array {
    $stmt = $this->pdo->prepare('
        SELECT * FROM studies 
        WHERE user_id = ? 
        AND title LIKE ?
        ORDER BY created DESC
    ');
    $stmt->execute([$userId, '%' . $query . '%']);
    $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
    return array_map([$this, 'mapToDomain'], $rows);
}
```

## Creating a New Repository

### Step 1: Create Repository File

```bash
touch App/Data/Repositories/MyRepository.php
```

```php
<?php
declare(strict_types=1);

namespace Application\App\Data\Repositories;

class MyRepository {
    private \PDO $pdo;
    
    public function __construct(\PDO $pdo) {
        $this->pdo = $pdo;
    }
    
    private function mapToDomain(?array $row): ?array {
        if (!$row) return null;
        return $row;
    }
    
    public function findById(int $id): ?array {
        $stmt = $this->pdo->prepare('SELECT * FROM my_table WHERE id = ?');
        $stmt->execute([$id]);
        return $this->mapToDomain($stmt->fetch(\PDO::FETCH_ASSOC));
    }
}
```

### Step 2: Register in Service

```php
class MyService {
    public function __construct(
        private MyRepository $repository
    ) {}
}
```

### Step 3: Wire in App.php

```php
private function createMyService(): MyService {
    return new MyService(
        new MyRepository($this->database->getConnection())
    );
}
```

## Best Practices

### 1. Always Use Prepared Statements

```php
// Good
$stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([$id]);

// Bad - SQL injection risk
$stmt = $this->pdo->query("SELECT * FROM users WHERE id = {$id}");
```

### 2. Map Database Rows to Domain

```php
private function mapToDomain(?array $row): ?array {
    if (!$row) return null;
    
    // Convert snake_case to camelCase if needed
    $row['createdAt'] = $row['created_at'];
    unset($row['created_at']);
    
    // Convert types
    $row['id'] = (int)$row['id'];
    $row['isActive'] = (bool)$row['is_active'];
    
    return $row;
}
```

### 3. Handle Null Results

```php
public function findById(int $id): ?array {
    $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
    $stmt->execute([$id]);
    $row = $stmt->fetch(\PDO::FETCH_ASSOC);
    
    // Return null if not found
    return $row ? $this->mapToDomain($row) : null;
}
```

### 4. Use Transactions for Multiple Operations

```php
public function transfer(int $fromId, int $toId, float $amount): bool {
    try {
        $this->pdo->beginTransaction();
        
        // Deduct from sender
        $stmt = $this->pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?');
        $stmt->execute([$amount, $fromId]);
        
        // Add to receiver
        $stmt = $this->pdo->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?');
        $stmt->execute([$amount, $toId]);
        
        $this->pdo->commit();
        return true;
    } catch (\Exception $e) {
        $this->pdo->rollBack();
        return false;
    }
}
```

### 5. Consistent Method Naming

| Operation | Method Name |
|-----------|-------------|
| Find by ID | `findById(int $id): ?array` |
| Find by column | `findByEmail(string $email): ?array` |
| Get all | `getAll(int $limit, int $offset): array` |
| Get by foreign key | `getByUserId(int $userId): array` |
| Count | `countAll(): int` |
| Save | `save(array $data): int` |
| Delete | `delete(int $id): bool` |

## Testing Repositories

### Unit Test with In-Memory SQLite

```php
<?php
require_once __DIR__ . '/../../vendor/autoload.php';

use Application\App\Data\Repositories\MyRepository;

// Create in-memory database
$pdo = new PDO('sqlite::memory:');
$pdo->exec('
    CREATE TABLE my_table (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
');

$repo = new MyRepository($pdo);

// Test insert
$id = $repo->save(['name' => 'Test Item']);
assert($id > 0);

// Test find
$item = $repo->findById($id);
assert($item['name'] === 'Test Item');

// Test update
$repo->save(['id' => $id, 'name' => 'Updated']);
$item = $repo->findById($id);
assert($item['name'] === 'Updated');

// Test delete
$repo->delete($id);
$item = $repo->findById($id);
assert($item === null);

echo "✓ All tests passed\n";
```

## Common Pitfalls

### 1. N+1 Queries

```php
// Bad - N+1 queries
$studies = $repo->getAll();
foreach ($studies as $study) {
    $user = $userRepo->findById($study['user_id']); // Query in loop!
}

// Good - Single query with JOIN
$studies = $repo->getAllWithUser();
```

### 2. Not Handling Null

```php
// Bad - May cause errors
$item = $repo->findById(999);
echo $item['name']; // Error if not found

// Good - Check for null
$item = $repo->findById(999);
if ($item) {
    echo $item['name'];
}
```

### 3. SQL Injection

```php
// Bad - Never do this
$stmt = $pdo->query("SELECT * FROM users WHERE name = '{$name}'");

// Good - Always use prepared statements
$stmt = $pdo->prepare('SELECT * FROM users WHERE name = ?');
$stmt->execute([$name]);
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
