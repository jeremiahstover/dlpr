# Public Pages

Public Pages provide a 3-stage fallback system for serving marketing pages, landing pages, and other public content through the `/page/*` URL pattern. This system allows content creators to add pages without modifying application code.

## Overview

The Public Pages system serves content at URLs like `/page/about`, `/page/features`, `/page/contact`. It uses a cascading fallback mechanism to find the appropriate content based on user authentication status and interface level.

## URL Structure

```
/page/{page-name}

Examples:
- /page/about      → Public/Pages/about.html
- /page/features   → Public/Pages/features.html
- /page/contact    → Public/Pages/contact.html
```

## 3-Stage Fallback System

### For Guest Users (Not Authenticated)

Only Stage 3 is available:

```
Stage 3: Public/Pages/{name}.html
```

### For Authenticated Users

All three stages are checked in order:

```
Stage 1: App/Presentation/{interface}/page-show.html
    ↓ (not found)
Stage 2: App/Presentation/page-show.html
    ↓ (not found)
Stage 3: Public/Pages/{name}.html
    ↓ (not found)
404 Error
```

## Stage Details

### Stage 1: Interface-Specific Template (Authenticated Only)

**Location:** `App/Presentation/{interface}/page-show.html`

Used when you want different layouts for different user interface levels (focused, normal, power).

```html
<!-- App/Presentation/focused/page-show.html -->
<div class="page-content focused-layout">
    <h1><?= htmlspecialchars($title) ?></h1>
    <div class="content">
        <?= $htmlContent ?>
    </div>
</div>
```

### Stage 2: Default Template (Authenticated Only)

**Location:** `App/Presentation/page-show.html`

The default template used when no interface-specific template exists.

```html
<!-- App/Presentation/page-show.html -->
<div class="page-content">
    <h1><?= htmlspecialchars($title) ?></h1>
    <div class="content">
        <?= $htmlContent ?>
    </div>
</div>
```

### Stage 3: Static HTML Content (All Users)

**Location:** `Public/Pages/{name}.html`

Plain HTML files containing the actual page content. These are inserted into the template.

```html
<!-- Public/Pages/about.html -->
<section>
    <h2>About MemorizeLive</h2>
    <p>Our mission is to help you memorize scripture...</p>
</section>
```

## Directory Structure

```
Public/Pages/
├── README.md           # Documentation for content editors
├── home.html           # Homepage content
├── about.html          # About page content
├── features.html       # Features page content
├── contact.html        # Contact page content
└── ...                 # Additional pages

App/Presentation/
├── page-show.html      # Default page template (Stage 2)
└── focused/
    └── page-show.html  # Focused interface template (Stage 1)
```

## Adding a New Public Page

### Step 1: Create the HTML Content

Create a new HTML file in `Public/Pages/`:

```bash
# Create the page content file
touch Public/Pages/my-page.html
```

```html
<!-- Public/Pages/my-page.html -->
<section class="hero">
    <h2>Welcome to My Page</h2>
    <p>This is the content of my new page.</p>
</section>

<section class="details">
    <h3>More Information</h3>
    <p>Additional content here...</p>
</section>
```

### Step 2: Access the Page

The page is immediately available at:

```
https://yoursite.com/page/my-page
```

### Step 3: Add to Navigation (Optional)

Add the page to the menu via `Data/menu.json`:

```json
{
    "menu": [
        {
            "label": "My Page",
            "url": "/page/my-page"
        }
    ]
}
```

Or create an alias in `Data/aliases.json`:

```json
{
    "/mypage": "/page/my-page"
}
```

## Security Considerations

### Path Traversal Prevention

The system sanitizes page names using `basename()`:

```php
$page = basename($page); // Strips path components
```

This prevents access to files outside `Public/Pages/`.

### Authentication Check

Guest users can only access Stage 3 (Public/Pages). The system checks authentication before attempting to load interface-specific templates.

### Content Escaping

Page content is rendered unescaped in templates. **Do not include user-generated content in Public/Pages files.**

## SEO Best Practices

### Meta Tags

Add meta tags to the page content:

```html
<!-- Public/Pages/about.html -->
<meta name="description" content="Learn about MemorizeLive's mission to help you memorize scripture.">
<meta name="keywords" content="bible, memorization, scripture, learning">

<section>
    <h1>About Us</h1>
    ...
</section>
```

### Semantic HTML

Use proper heading hierarchy:

```html
<!-- Good -->
<h1>Main Title</h1>
<h2>Section Heading</h2>
<h3>Subsection</h3>

<!-- Bad -->
<h3>Main Title</h3>
<h1>Section</h1>
```

### Clean URLs

Use aliases for cleaner URLs:

```json
// Data/aliases.json
{
    "/about": "/page/about",
    "/features": "/page/features",
    "/contact": "/page/contact"
}
```

## Testing Pages

### Local Development

```bash
# Start the development server
./serve.sh

# Test the page
curl http://localhost:8080/page/my-page
```

### Verify Rendering

```bash
# Check as guest (Stage 3 only)
curl http://localhost:8080/page/about

# Check as authenticated user (Stages 1-3)
# (Log in via browser, then access /page/about)
```

### Test Aliases

```bash
# Test alias resolution
curl http://localhost:8080/about
# Should redirect/render as /page/about
```

## Common Patterns

### Hero Section

```html
<section class="hero">
    <div class="container">
        <h1>Page Title</h1>
        <p class="subtitle">Compelling subtitle text</p>
        <a href="/signup" class="button primary">Get Started</a>
    </div>
</section>
```

### Feature Grid

```html
<section class="features">
    <div class="grid">
        <article>
            <h3>Feature One</h3>
            <p>Description of feature one.</p>
        </article>
        <article>
            <h3>Feature Two</h3>
            <p>Description of feature two.</p>
        </article>
    </div>
</section>
```

### Call to Action

```html
<section class="cta">
    <div class="container">
        <h2>Ready to Start?</h2>
        <p>Join thousands of users memorizing scripture.</p>
        <a href="/register" class="button large">Sign Up Free</a>
    </div>
</section>
```

## Troubleshooting

### 404 Error

- Verify file exists: `Public/Pages/{name}.html`
- Check file permissions (readable)
- Ensure no path traversal in page name

### Wrong Template Loading

- Check if authenticated (affects Stage 1-2 availability)
- Verify interface level for Stage 1
- Check for template file existence

### Styling Issues

- Public pages use the same CSS as the main app
- Add custom styles to `Public/Assets/css/`
- Use existing CSS classes from the theme

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
