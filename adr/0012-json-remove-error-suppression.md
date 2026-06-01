# ADR-0012: Remove Error Suppression During Getter Calls

**Date:** 2026-06-02
**Status:** Proposed

## Context

In `JsonConverter::objectToJson()`, the library nullifies PHP's error handler before calling getter methods, then restores it:

```php
set_error_handler(null);

for ($y = 0 ; $y < $count; $y++) {
    $funcNm = substr($methods[$y], 0, 3);

    if (strtolower($funcNm) == 'get') {
        $propVal = call_user_func([$obj, $methods[$y]]);
        // ...
    }
}

restore_error_handler();
```

This suppresses all errors from getters that:
- Require arguments (`getItemAt(int $index)`) → TypeError
- Access uninitialized typed properties → Error
- Trigger deprecation notices
- Have side effects that fail

The property silently disappears from JSON output with no indication of failure. Developers see missing fields and have no clue why.

## Decision

Remove `set_error_handler(null)`. Instead, use reflection to check if the method has zero required parameters before calling it:

```php
$refMethod = new \ReflectionMethod($obj, $methods[$y]);
if ($refMethod->getNumberOfRequiredParameters() === 0) {
    $propVal = $refMethod->invoke($obj);
    $json->add(substr($methods[$y], 3), $propVal);
}
```

This is precise — only calls actual zero-argument getters — and lets real errors propagate naturally to the developer.

## Alternatives Considered

- **Keep suppression, document the behavior** — does not solve the debugging problem.
- **Wrap each call in try/catch and skip on exception** — still hides errors, just more explicitly.
- **Only suppress specific error types (TypeError)** — complex, still masks some legitimate failures.

## Consequences

- **Breaking change**: methods like `getItemAt(int $index)` that previously failed silently will now simply not be called (correct behavior). Methods that throw on uninitialized state will now propagate the error to the caller (desired for debugging).
- **Eliminates silent failures**: missing properties in JSON output become diagnosable.
- **Slightly more reflection overhead**: one `ReflectionMethod` per getter. Acceptable given the library already uses reflection for public properties.
- **Pairs with `#[JsonIgnore]`** ([ADR-0009](0009-json-include-null-false-and-json-ignore.md)): developers can now explicitly exclude methods rather than relying on error suppression to skip them.

## GitHub Issue

[webfiori/json#60](https://github.com/WebFiori/json/issues/60)
