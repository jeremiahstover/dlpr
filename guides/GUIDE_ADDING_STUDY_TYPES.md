# Guide: Adding New Study Types

Add support for new study types beyond 'reference', 'topic', and 'collection'.

## Overview

Study types determine how references are collected and displayed. Each type has:
- Form input handling (search, selection, or direct entry)
- Backend validation and storage
- Template rendering variations (optional)

---

**Status:** Accurate and Complete
**Last Updated:** 2025-02-08

## Step 1: Add Type to Database

No schema change needed - `type` column is TEXT.

Just ensure new type name is consistent everywhere.

## Step 2: Update Form Template

**File:** `App/Presentation/Studies/form.php`

Add type selector option:
```php
<select name="type" id="study-type">
    <option value="verse">Verse</option>
    <option value="topic">Topic</option>
    <option value="collection">Collection</option>
    <option value="custom">Custom Type</option>  <!-- NEW -->
</select>
```

Add reference input section for new type:
```php
<div id="reference-custom-section" class="reference-section" style="display: none;">
    <label>Enter your reference</label>
    <input type="text" name="reference" id="ref-custom" placeholder="...">
</div>
```

## Step 3: Add JavaScript Handler

**File:** `App/Presentation/Studies/form.php` (inline script section)

The form uses inline JavaScript for type handling. Add your new type to the existing switch statements:

### Update the change handler:
```javascript
// Around line 219 - Add to radio button change handler
if (type === 'custom') {
    document.getElementById('field-custom')?.classList.remove('is-hidden');
    document.getElementById('field-custom')?.classList.add('is-visible');
}
```

### Update title generation:
```javascript
// Around line 260 - Add to title generation logic
if (type === 'custom') {
    const customRef = document.getElementById('ref-custom')?.value || '';
    title = `Custom: ${customRef || 'Study'}`;
}
```

### Update reference retrieval:
```javascript
// Around line 268 - Add to reference collection
if (type === 'custom') {
    reference = document.getElementById('ref-custom')?.value || '';
}
```

**Note:** The old `study-form.js` file contained these functions but they have been migrated to inline scripts in the form template for better performance and simpler dependency management.

## Step 4: Update Validation

**File:** `App/Routes/Controllers/StudyController::save()`

Add validation for new type:
```php
switch ($type) {
    case 'verse':
        // Validate verse format
        break;
    case 'topic':
        // Validate topic exists
        break;
    case 'collection':
        // Validate collection ID
        break;
    case 'custom':
        // Custom validation rules
        if (empty($reference)) {
            $errors['reference'] = 'Reference is required';
        }
        break;
}
```

## Step 5: Update List View (Optional)

**File:** `App/Presentation/Studies/list.php`

Add type-specific icon or badge:
```php
<?php foreach ($studies as $study): ?>
    <div class="study-item">
        <?php if ($study['type'] === 'custom'): ?>
            <span class="tag is-info">Custom</span>
        <?php endif; ?>
        <!-- ... -->
    </div>
<?php endforeach; ?>
```

## Step 6: Add Tests

**File:** `Tests/test-study-types.php`

```php
// Test custom type creation
$response = $controller->save($app, [
    'post' => [
        'title' => 'My Custom Study',
        'type' => 'custom',
        'reference' => 'My custom content'
    ]
]);
assert contains validation for custom type
assert stores reference correctly
```

## Testing Checklist

- [ ] Form shows new type option
- [ ] Selecting type shows correct input section
- [ ] Reference saves correctly
- [ ] Title auto-generates appropriately
- [ ] Validation works for edge cases
- [ ] List view displays type correctly
- [ ] API search works (if applicable)

## Pattern Reference

| Component | File | Example |
|-----------|------|---------|
| Type selector | form.php | `<select name="type">` |
| Input section | form.php | `<div id="reference-X-section">` |
| Section toggle | study-form.js | `toggleReferenceSection()` |
| Title gen | study-form.js | `generateStudyTitle()` |
| Reference get | study-form.js | `getSelectedReference()` |
| Validation | StudyController | `save()` method |
