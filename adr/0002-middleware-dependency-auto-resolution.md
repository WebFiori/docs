# ADR-0002: Middleware Dependency Auto-Resolution

**Date:** 2026-05-31
**Status:** Accepted

## Context

Middleware can declare dependencies via `getDependencies()`. Currently, these dependencies are only used for **sorting** middleware that are already assigned to a route. If a dependency is not explicitly assigned, it is silently skipped — it does not get pulled in automatically.

Example: if E depends on D, D on C, C on B, B on A:

```php
// Only E runs. D, C, B, A are silently skipped.
RouteOption::MIDDLEWARE => ['mw-e']
```

This makes `getDependencies()` behave as an execution order hint rather than a true dependency declaration. Developers must manually list the entire chain, defeating the purpose of declaring dependencies.

## Decision

Add a `resolveDependencies()` step in `RouterUri::getMiddleware()` that walks the dependency graph via BFS and pulls in missing middleware from the `MiddlewareRegistry` before sorting:

```php
public function getMiddleware(): array {
    if (count($this->assignedMiddlewareList) != count($this->sortedMiddleeareList)) {
        $resolved = $this->resolveDependencies($this->assignedMiddlewareList);
        $this->sortedMiddleeareList = $this->sortByDependencies($resolved);
    }

    return $this->sortedMiddleeareList;
}

private function resolveDependencies(array $middlewareList): array {
    $byName = [];

    foreach ($middlewareList as $mw) {
        $byName[$mw->getName()] = $mw;
    }

    $queue = $middlewareList;

    while (!empty($queue)) {
        $current = array_shift($queue);

        foreach ($current->getDependencies() as $depName) {
            if (isset($byName[$depName])) {
                continue;
            }

            $dep = MiddlewareManager::getMiddleware($depName);

            if ($dep !== null) {
                $byName[$depName] = $dep;
                $queue[] = $dep;
            }
        }
    }

    return array_values($byName);
}
```

**Execution order** is determined by the existing `sortByDependencies()` (Kahn's algorithm):
- Dependency order takes precedence (A before B if B depends on A).
- Priority is used as tiebreaker only for middleware with no dependency relationship.
- Circular dependencies are detected and throw `RoutingException`.

**Cycle safety**: `$byName` acts as a visited set — each middleware is processed at most once, so the BFS queue always drains. The existing `sortByDependencies()` provides a second layer of cycle detection.

## Alternatives Considered

- **Keep current behavior, document it** — but this defeats the purpose of `getDependencies()` and forces developers to maintain dependency knowledge manually.
- **Throw an error when a dependency is not assigned** — more explicit but less ergonomic. Developers would still need to list the full chain.
- **Auto-resolve only one level (no transitive)** — simpler but incomplete. A depends on B depends on C would still require manual listing of C.

## Consequences

- **Non-breaking**: middleware with no dependencies behaves exactly as before.
- **Transitive**: if E depends on D which depends on C, assigning E pulls in D and C.
- **Only pulls from registered middleware**: if a dependency isn't registered in `MiddlewareManager`, it's silently skipped (same as current behavior).
- **Efficient**: resolution runs once per route, result is cached in `$sortedMiddleeareList`.
- **Trade-off**: middleware that was previously silently skipped will now execute. This is the correct behavior but could surface latent bugs in applications that accidentally declared dependencies they didn't intend to run.

GitHub Issue: https://github.com/WebFiori/framework/issues/380
