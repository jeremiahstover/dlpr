# Route Mapping - UserController Split

## Overview
All 22 user-related routes have been successfully migrated from the monolithic UserController to four domain-specific controllers.

## Route Distribution

### 🔐 IdentityController (8 routes)
Authentication, signup, login, and email validation flows

| Route | Method | Purpose |
|-------|--------|---------|
| `/user/login` | login() | Login form and processing |
| `/user/logout` | logout() | Logout handling |
| `/user/signup` | signup() | User signup form/processing |
| `/user/send-login-link` | sendLoginLink() | Send magic login link via email |
| `/validate-login` | validateLogin() | Validate magic login link |
| `/validate` | validate() | Email validation/verification |
| `/user/generate-validation-link` | generateValidationLink() | Send verification email |
| `/user/suppress-timezone-warning` | suppressTimezoneWarning() | Suppress timezone warning |

### 👤 ProfileController (11 routes)
User profile, settings, password, configuration, and focused times management

| Route | Method | Purpose |
|-------|--------|---------|
| `/user[/{id}]` | view() | View user profile |
| `/user[/{id}]/edit` | edit() | Combined edit form |
| `/user[/{id}]/edit-profile` | editProfile() | Edit profile fields only |
| `/user/save-profile` | saveProfile() | Save profile updates |
| `/user[/{id}]/edit-password` | editPasswordForm() | Password change form |
| `/user/save-password` | savePassword() | Save password change |
| `/user[/{id}]/edit-config` | editConfig() | Edit user configuration |
| `/user/save-config` | saveConfig() | Save configuration |
| `/user[/{id}]/edit-focused-times` | editFocusedTimes() | Edit focused times |
| `/user/save-focused-times` | saveFocusedTimes() | Save focused times |
| `/user/save` | save() | Legacy delegator (→ saveFocusedTimes) |

### 🎨 MenuCustomizationController (2 routes)
Menu override customization management

| Route | Method | Purpose |
|-------|--------|---------|
| `/user[/{id}]/edit-menu-overrides` | editMenuOverrides() | Edit menu overrides form |
| `/user/save-menu-overrides` | saveMenuOverrides() | Save menu customizations |

### 👨‍💼 AdminUserController (1 route)
Admin-specific user management operations

| Route | Method | Purpose |
|-------|--------|---------|
| `/users` | list() | List all users with pagination |

## Route Aliases (from aliases.json)

The following route aliases continue to work unchanged:

| Alias | Maps To | Controller |
|-------|---------|------------|
| `/login` | `/user/login` | IdentityController |
| `/logout` | `/user/logout` | IdentityController |
| `/signup` | `/user/signup` | IdentityController |
| `/profile` | `/user` | ProfileController |
| `/profile[/{id}]` | `/user[/{id}]` | ProfileController |
| `/profile[/{id}]/edit` | `/user[/{id}]/edit` | ProfileController |
| `/profile/save` | `/user/save` | ProfileController |

## Statistics

### Total Routes: 22
- IdentityController: 8 routes (36%)
- ProfileController: 11 routes (50%)
- MenuCustomizationController: 2 routes (9%)
- AdminUserController: 1 route (5%)

### Route Patterns
- **Public routes** (no auth): `/user/login`, `/user/signup`, `/validate-login`, `/validate`
- **Authenticated routes**: All other routes require authentication
- **Admin routes**: `/users` requires admin role
- **Ownership routes**: Routes with `[/{id}]` support both own profile and admin access

## Verification

All routes have been verified:
- ✅ Routes.json is valid JSON
- ✅ All controllers exist and compile
- ✅ All methods exist in their respective controllers
- ✅ No duplicate route definitions
- ✅ All aliases continue to work

## Migration Impact

### No Breaking Changes
- ✅ All existing URLs continue to work
- ✅ All form actions remain unchanged
- ✅ All template references remain valid
- ✅ All middleware integrations preserved

### Improved Organization
- ✅ Clear domain separation
- ✅ Easier to locate route handlers
- ✅ Reduced cognitive load for developers
- ✅ Better testability per domain

---

**Generated:** January 5, 2025
**Total Routes Migrated:** 22
**Controllers Created:** 4
**Breaking Changes:** 0

---

**Note:** This document records a completed migration (January 2025). The UserController was split into four domain-specific controllers. All routes continue to function as documented.

**Related Docs:**
- [ROUTE_ALIASES.md](ROUTE_ALIASES.md) - Route alias system
- [URI_TYPES_OVERVIEW.md](URI_TYPES_OVERVIEW.md) - URI type classification
- [ACCESS_CONTROL_OVERVIEW.md](ACCESS_CONTROL_OVERVIEW.md) - Access control for routes

**Document Version:** 1.0
**Last Updated:** 2025-02-08
**Status:** Accurate and Complete (Historical Record)
