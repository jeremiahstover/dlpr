# Response Patterns

Consistent response patterns ensure that the frontend (whether standard HTML, HTMX, or JSON clients) can reliably handle the results of controller actions.

## Standard Success Pattern
Success responses should generally return the data requested or a confirmation message.

```php
return ResponseBuilder::success(
    data: ['id' => $new_id],
    message: 'Resource created successfully',
    template: 'Partials/success_message'
);
```

## Standard Error Pattern
Error responses should always include a human-readable message and an appropriate HTTP status code.

```php
return ResponseBuilder::error(
    message: 'Invalid input provided',
    status: 422,
    template: 'Partials/error_display'
);
```

## Custom Error Views
By using the `$template` parameter in error methods, we can provide rich error experiences:

```php
// Custom 404 page
return ResponseBuilder::notFound(
    message: 'The requested study could not be found.',
    template: 'Errors/study_not_found'
);

// Custom forbidden page
return ResponseBuilder::forbidden(
    message: 'You do not have permission to view this study.',
    template: 'Errors/access_denied'
);
```

## HTMX vs Standard Requests
The response builder structure allows the application to differentiate between full page loads and partial updates:

1. **JSON Request**: Returns the array as JSON.
2. **HTMX Request**: If a template is provided, renders the template (often a partial) and returns HTML.
3. **Standard Request**: If a template is provided, renders the template within the main layout.

This flexibility is maintained by passing the `$template` parameter through the standard `ResponseBuilder` methods.
