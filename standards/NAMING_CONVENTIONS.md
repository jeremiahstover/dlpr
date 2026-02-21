# Naming Conventions

This document defines the naming conventions for the `application` repository to ensure consistency across the codebase.

## Routes
All routes must be all lowercase and use kebab-case for multiple words.
- ✅ `/user/edit-profile`
- ✅ `/studies/{id}/send-now`
- ❌ `/user/editProfile`
- ❌ `/User/Edit-Profile`

## Database
- **Tables**: Use snake_case plural nouns (e.g., `notifications`, `users`).
- **Columns**: Use snake_case singular nouns (e.g., `first_name`, `email_address`).
- **Timestamps**: All tables that track creation and updates must use exactly `created` and `updated` as column names.

## Domain Data & Repository Mapping
When mapping database records to domain arrays in repositories, follow this pattern:
- DB `created` → Array key `created_at`
- DB `updated` → Array key `updated_at`

Example Repository Mapping:
```php
return [
    'id' => $row['id'],
    'title' => $row['title'],
    'created_at' => $row['created'],
    'updated_at' => $row['updated'],
];
```

## PHP Standards
We follow PSR-12 coding standards with the following specific rules:

- **Classes**: `PascalCase` (e.g., `StudyService`, `NotificationRepository`).
- **Methods**: `camelCase` (e.g., `getStudyById`, `saveNotification`).
- **Variables**: `snake_case` (e.g., `$user_id`, `$study_data`).
- **Array Keys**: `snake_case` (e.g., `['user_id' => 123]`).
- **Constants**: `SCREAMING_SNAKE_CASE` (e.g., `STATUS_ACTIVE`).
