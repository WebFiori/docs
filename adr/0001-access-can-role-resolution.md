# ADR-0001: Access::can() Role Resolution Strategy

**Date:** 2026-05-31
**Status:** Accepted

## Context

`AccessManager::can($user, $permission, $resource)` resolves the user's roles by calling `$this->getUserRoles($userId)`, which looks up an internal map populated only by `Access::assignRoleToUser()`.

When the `$user` object implements `SecurityPrincipal` (which provides `getRoles()`), the method ignores it. This forces developers to keep two sources of truth in sync:

```php
SecurityContext::setCurrentUser($user);          // has getRoles()
Access::assignRoleToUser($user->getId(), $role); // populates internal map
```

Forgetting `assignRoleToUser()` causes `can()` to silently return `false` even though the user clearly has the role.

## Decision

Use the internal `$userRoles` map as the primary source. If no roles are found in the map for the given user, fall back to `$user->getRoles()` when the user object implements the method.

```php
public function can($user, string $permission, ?object $resource = null): bool {
    $userId = is_object($user) && method_exists($user, 'getId') ? $user->getId() : $user;
    $roles = $this->getUserRoles($userId);

    if (empty($roles) && is_object($user) && method_exists($user, 'getRoles')) {
        $roles = $user->getRoles();
    }

    // ... rest of RBAC + ABAC check
}
```

## Alternatives Considered

- **User object always takes precedence over internal map** — cleaner single source of truth, but breaks existing code that relies on `assignRoleToUser()` to override or augment roles at runtime.
- **No change, document the requirement** — keeps the status quo but leaves a common footgun for new developers.

## Consequences

- **Non-breaking**: existing code using `assignRoleToUser()` continues to work unchanged.
- **Eliminates silent failures**: passing a `SecurityPrincipal` to `can()` works without extra setup.
- **`assignRoleToUser()` remains useful** for scenarios where roles come from database storage or need runtime override.
- **Trade-off**: two code paths for role resolution — could cause confusion if both are populated with different values. The internal map wins in that case, which is the safer default.

## Related

- [ADR-0003](0003-requires-auth-security-context.md) — same root issue: framework not fully trusting `SecurityContext` as the auth/authz source of truth.

## GitHub Issue

[webfiori/framework#381](https://github.com/WebFiori/framework/issues/381)
