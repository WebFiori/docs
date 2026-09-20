# ADR-0050: HTTP: Per-Class Caching of WebService Annotation Configuration

**Date:** 2026-09-21
**Status:** Accepted

## Context

`WebFiori\Http\WebService` derives its routing, parameter, and authentication
configuration from PHP attributes (`#[RestController]`, `#[GetMapping]`,
`#[RequestParam]`, `#[RequiresAuth]`, `#[PreAuthorize]`, `#[UseParameterSet]`,
etc.) via reflection. This reflection ran on **every** relevant operation:

- **Construction:** `configureFromAnnotations()` created multiple
  `ReflectionClass` instances and, for every method, checked five mapping
  attributes — O(methods × attributes) — recomputing data that is static per
  class. This is a hot path: every request constructs at least one service, and
  a `WebServicesManager` eagerly constructs **all** registered services up front.
- **Per request/method:** `configureParametersForMethod()` re-read
  `RequestParam`/`UseParameterSet` attributes, and `checkMethodAuthorization()`
  re-read the auth attributes, on each call.

A class's annotations are immutable for the lifetime of a process, so all of this
recomputes identical results. See issue #154 (which supersedes the earlier #50/#78
discussion for `WebService` specifically).

## Decision

Cache the **derived, class-static** annotation configuration per concrete class
(keyed by `static::class`), computing it once per class per process and replaying
it on subsequent constructions/calls without reflection. Three caches are added
to `WebService`, all `private static`:

1. **`$annotationCache`** — constructor-path config: `path`, `description`,
   `requestMethods`, `authRequired`, plus whether the class is annotated and its
   annotation name.
2. **`$methodParamCache`** — per `(class, method)` parameter descriptors: the
   `UseParameterSet` class names and the `RequestParam` option arrays.
3. **`$methodAuthCache`** — per `(class, method)` authorization *facts*:
   attribute presence flags and the `PreAuthorize` expression **string**.

### The name is resolved per instance, never cached

The service **name** is intentionally excluded from the cache. `WebService` may be
constructed directly with different names for the same class
(`new WebService('users')` vs `new WebService('orders')`), so caching a resolved
name by class would corrupt subsequent instances. Instead, only the *annotation
name* (if the class is annotated) is cached; the final name is resolved on each
construction from the annotation name or the constructor's fallback argument.

### Cache is recorded only after the reflection path succeeds

`configureMethodMappings()` can throw `DuplicateMappingException`. The cache entry
is recorded **only after** the full miss-path (including that check) completes, so
a throwing class never caches a bad state — it re-throws on every construction,
identical to prior behavior.

### Authorization decisions are never cached

`checkMethodAuthorization()` mixes static facts with live `SecurityContext` calls
(`isAuthenticated()`, `evaluateExpression()`). Only the static facts are cached;
`SecurityContext` is always evaluated at call time, so authorization outcomes are
never frozen.

### Parameter objects are rebuilt per instance (OpenAPI-safe)

`configureParametersFromMethod()` mutates the instance (`addParameter`,
`addParameterSet`). Only plain descriptors (scalars/arrays and parameter-set class
names) are cached; the actual parameter mutations are **replayed per instance**
with fresh objects. No mutable object is shared between instances, which keeps
OpenAPI generation — which reads per-service, per-method metadata — correct and
deterministic.

## Alternatives Considered

**Caching the resolved name too (the original prototype in #154).** Simpler, but
incorrect: it breaks direct `new WebService($name)` construction where the same
class is used with different names. Rejected.

**Caching parameter/response objects instead of descriptors.** A slightly larger
speedup, but sharing mutable objects across instances risks corrupting OpenAPI
output and per-request state. Rejected in favor of rebuilding per instance.

**Lazy service registration in `WebServicesManager`** (construct only the service
that handles the request). Helps the manager case but does not remove per-service
reflection for the service that runs, and changes the registration API. Complementary
and lower priority; can be added later.

**Doing nothing / documenting the cost.** The cost is on a universal hot path and
scales with service and method counts; a transparent code fix is preferable.

## Consequences

**Easier / better:**
- Large reduction in per-request reflection. Measured on a param-heavy service
  (clean back-to-back A/B, `php -n -d opcache.enable_cli=1`, Xdebug off, 50k iters):
  single construction ~12×, manager + 6 services ~10×, per-method parameter
  configuration (fresh instance) ~6×, per-method authorization (fresh instance) ~5×.
  (Absolute microseconds are machine-specific; the ratios are the portable takeaway.)
- No public API change; fully transparent to callers.
- OpenAPI generation, authorization, duplicate-mapping detection, and unannotated
  fallback-name behavior are unchanged (verified by the existing suite plus new
  equivalence tests).

**Harder / trade-offs:**
- Introduces `private static` process-level caches. Correct for PHP's
  shared-nothing request model, and bounded by the number of service classes.
- Contributors must preserve the invariants: never cache the name for
  unannotated classes, never cache authorization decisions, rebuild mutable
  parameter state per instance, and record the constructor cache only after the
  duplicate-mapping check.
