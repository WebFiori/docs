# ADR-0015: Only Support `get*` Prefix for Auto-Mapping Object Methods

**Date:** 2026-06-02
**Status:** Accepted

## Context

When auto-mapping objects to JSON, the library only calls methods prefixed with `get`:

```php
if (strtolower($funcNm) == 'get') {
    $propVal = call_user_func([$obj, $methods[$y]]);
}
```

Boolean accessors like `isActive()`, `hasChildren()`, `canEdit()` are not detected. The question is whether to add support for `is*`, `has*`, and `can*` prefixes.

## Decision

Keep `get*` as the only supported prefix. Developers should use `getIsActive()`, `getHasChildren()`, etc. for boolean properties they want serialized.

### Arguments for adding `is*/has*/can*` (rejected)

1. **Matches JavaBeans convention** — Java's `BooleanProperty` uses `is*` prefix.
2. **More natural PHP naming** — `isActive()` reads better than `getIsActive()`.
3. **Other serializers support it** — Jackson, Symfony Serializer detect `is*`.

### Arguments against (accepted)

1. **Ambiguity in name derivation.** `getName()` → strip `get` → `Name` is clear. But `isActive()` → strip `is` → `Active`? Or `isActive`? Or `is_active`? What about `isEmpty()` on a collection — is that a property or a state check? The mapping is less obvious and creates inconsistency.

2. **Scope creep.** Today it's `is*`/`has*`/`can*`. Tomorrow someone asks for `should*`, `will*`, `was*`. Where do you stop? With `get*` only, the boundary is clear and permanent.

3. **Increased `#[JsonIgnore]` burden.** More prefixes means more methods auto-called, which means more things developers need to explicitly exclude. `hasNext()` on an iterator, `isEmpty()` on a collection, `canRetry()` on an HTTP client — all would need `#[JsonIgnore]` to prevent unintended serialization.

4. **`getIsActive()` works fine.** It's slightly more verbose but completely unambiguous. The developer explicitly opts in to serialization by naming a method `get*`. No surprises, no accidental serialization.

5. **Reflection check doesn't fully protect.** Even with zero-argument filtering ([ADR-0012](0012-json-remove-error-suppression.md)), `is*`/`has*` methods are often behavioral checks (not property accessors) that happen to take no arguments. `isEmpty()`, `isConnected()`, `hasTimedOut()` — these are state queries, not data properties.

6. **Clear contract.** `get*` is a single, memorable rule: "prefix with `get` = this gets serialized." Adding more prefixes makes the contract fuzzy and increases the surface area for unintended serialization. The cost of `getIsActive()` over `isActive()` is trivial compared to the debugging cost of "why is `hasNext` appearing in my JSON?"

## Consequences

- **Simple rule**: one prefix to remember, one prefix to document.
- **No accidental serialization**: behavioral methods (`isEmpty`, `hasNext`, `canRetry`) are never called.
- **Slightly verbose for booleans**: `getIsActive()` instead of `isActive()`. This is an acceptable trade-off for clarity.
- **Developers retain full control**: combined with `#[JsonIgnore]` for exclusion and `#[JsonProperty]` for naming, the `get*` convention provides a predictable baseline.

## GitHub Issue

N/A — documents existing intentional behavior.
