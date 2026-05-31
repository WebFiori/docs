# ADR-0003: #[RequiresAuth] Should Check SecurityContext Directly

**Date:** 2026-05-31
**Status:** Accepted

## Context

The `#[RequiresAuth]` attribute on a service method is intended to mean "this endpoint requires an authenticated user." However, the current implementation delegates to `$this->isAuthorized()`, which defaults to `false`.

This default exists for a reason: in the traditional (non-attribute) API style, `isAuthorized()` returning `false` acts as a secure default — deny unless the developer explicitly overrides it with custom logic.

The conflict: when using the attribute-based style, `#[RequiresAuth]` should check authentication state from `SecurityContext`, not call a method designed for a different API style.

The auth evaluation flow for a method is:
1. `#[AllowAnonymous]` → allow (skip auth)
2. `#[RequiresAuth]` → **currently calls `isAuthorized()`** → defaults to `false`
3. `#[PreAuthorize]` → evaluate expression
4. No attributes → call `isAuthorized()` (secure default)

## Decision

When `#[RequiresAuth]` is present, check `SecurityContext::isAuthenticated()` directly. Do not call `isAuthorized()`. The `isAuthorized()` method remains the fallback only for methods with no auth attributes.

```php
if ($hasRequiresAuth) {
    if (!SecurityContext::isAuthenticated()) {
        return false;
    }

    // Then check PreAuthorize if present
    if (!empty($preAuthAttributes)) {
        return SecurityContext::evaluateExpression($preAuth->expression);
    }

    return true;
}

// No auth attributes → traditional fallback
return $this->isAuthorized();
```

Final auth flow:
- `#[AllowAnonymous]` → allow (unchanged)
- `#[RequiresAuth]` → `SecurityContext::isAuthenticated()`
- `#[PreAuthorize]` → evaluate expression (works with or without `#[RequiresAuth]`)
- No attributes → `isAuthorized()` (defaults to `false`, secure default)

## Alternatives Considered

- **Change default of `isAuthorized()` to `SecurityContext::isAuthenticated()`** — simpler but breaks services that intentionally use `false` as a deny-all default in the traditional style.
- **Document the override requirement** — leaves a footgun that every attribute-based service hits.

## Consequences

- **Non-breaking for traditional APIs**: services without auth attributes still use `isAuthorized()` with its `false` default.
- **Fixes attribute-based APIs**: `#[RequiresAuth]` works as its name implies without boilerplate overrides.
- **Clear separation**: attributes control auth for the attribute style; `isAuthorized()` controls auth for the traditional style. They don't interfere with each other.

GitHub Issue: https://github.com/WebFiori/http/issues/117
