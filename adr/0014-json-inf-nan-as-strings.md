# ADR-0014: Serialize INF/NaN as Strings (Never Crash During Serialization)

**Date:** 2026-06-02
**Status:** Accepted

## Context

`INF` and `NaN` are not valid JSON values per RFC 8259. Different languages handle them differently:

- PHP `json_encode()`: returns `false` (refuses to produce output)
- JavaScript `JSON.stringify()`: converts to `null`
- Python `json` module: throws `ValueError` by default

This library converts them to strings:

```php
if (is_nan($val)) {
    $retVal = '"NaN"';
} else if ($val == INF) {
    $retVal = '"Infinity"';
}
```

The output is valid JSON (`{"score":"NaN"}`), but the consumer receives a string where they might expect a number.

## Decision

Keep the current behavior: serialize `INF` as `"Infinity"` and `NaN` as `"NaN"` (string values).

The library's philosophy is **never crash during serialization**. A single edge-case numeric value should not abort the entire JSON output. The developer can inspect the output, see the string representation, and handle it appropriately.

## Alternatives Considered

- **Throw `JsonException`** — strict RFC compliance, but violates the "never crash" philosophy. One bad value in a large object graph would prevent any output.
- **Serialize as `null`** — type-safe (null is valid JSON), matches JavaScript's approach. But loses information — the developer can't distinguish "value was null" from "value was infinity."
- **Configurable strategy (throw/null/string)** — flexible but adds API complexity for an edge case that rarely occurs in practice.

## Consequences

- **Valid JSON output**: the string values are syntactically valid JSON.
- **Type mismatch for consumers**: a client expecting a number gets a string. This is a known trade-off — the developer is responsible for handling `INF`/`NaN` before serialization if strict typing matters.
- **No data loss**: the developer can see *what* the value was (`"Infinity"` vs `"NaN"` vs `null`) and act accordingly.
- **Consistent with library philosophy**: serialization is best-effort, never crashes.

## GitHub Issue

N/A — documents existing intentional behavior.
