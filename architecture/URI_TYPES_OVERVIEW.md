# URI Types Overview

## Overview

The Application application classifies incoming requests into five distinct URI types, each with its own processing pipeline and characteristics. This document provides an overview of the classification system.

## The Five URI Types

1. **Static Files** - CSS, JS, images, fonts served directly
2. **Public Lite Routes** - Public pages with 3-stage fallback
3. **Cron Routes** - Background job processing
4. **Heavy Routes** - Full application framework (authenticated)
5. **API Routes** - JSON API endpoints

## Classification Flow

```
Incoming Request
       ↓
Parse URI and Check Extension
       ↓
Is Static Asset? → Yes → Serve Static File
       ↓
      No
       ↓
Is /page/ Route? → Yes → Lite Processing
       ↓
      No
       ↓
Is /cron? → Yes → Cron Processing
       ↓
      No
       ↓
Default to Heavy Processing
       ↓
Check for /api/ Prefix → API Special Handling
```

## Performance Comparison

| Type | Latency | Memory | Authentication |
|------|---------|--------|----------------|
| Static | Lowest | Minimal | None |
| Lite | Low | Low | None |
| Cron | Medium | Medium | System |
| Heavy | High | High | Required |
| API | High | High | Token-based |

## Decision Criteria

### Static Files
- URI ends with: `.css`, `.js`, `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.ico`, `.woff`, `.woff2`, `.ttf`, `.eot`, `.json`
- Bypasses PHP framework entirely

### Lite Routes
- URI pattern: `/page/{name}` (no extension)
- Uses 3-stage fallback: HTML → PHP → Plates

### Cron Routes
- URI equals: `/cron`
- 3-stage execution: Notifications → Schedules → Studies

### Heavy Routes
- Default for all other URIs
- Full middleware stack, authentication required

### API Routes
- URI prefix: `/api/`
- Content negotiation: JSON responses

---

**See Also:**
- [URI_TYPE_STATIC.md](URI_TYPE_STATIC.md) - Static file handling
- [URI_TYPE_LITE.md](URI_TYPE_LITE.md) - Lite route processing
- [URI_TYPE_CRON.md](URI_TYPE_CRON.md) - Cron job processing
- [URI_TYPE_HEAVY.md](URI_TYPE_HEAVY.md) - Heavy route processing
- [URI_TYPE_API.md](URI_TYPE_API.md) - API route handling

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete