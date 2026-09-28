# ADR-0054: Per-Tool Outcome Reporting via Result Signal and Status Events

**Date:** 2026-09-29
**Status:** Accepted

## Context

When tools are auto-executed during `chat()`, the library emits status events
(`TOOL_CALLING`, `TOOL_EXECUTING`, `TOOL_COMPLETED`) so consumers — SSE streams,
callback handlers, and the `StatusMessageFormatter` — can render progress. Two
gaps existed:

1. **No success/failure signal.** `TOOL_COMPLETED` fired unconditionally with
   only `{tool, duration_ms}`. A tool "failure" was, by convention, an error
   *string* returned from `execute()` (e.g. `{"error": "..."}`) that the library
   passed through without inspecting. A failed tool call was therefore
   indistinguishable from a successful one in the event stream, so UIs showed a
   success checkmark for failures.

2. **`execute()` was not exception-guarded.** A throwing tool aborted the entire
   tool-call batch and surfaced as a request-level `ERROR`, rather than a
   per-tool failure that lets the remaining tools run.

ADR-0030 established exceptions (not result objects) as the error-handling
strategy for the library. Tool outputs are a distinct domain: a failed tool must
return a value the *model* can read and react to (so the conversation continues),
so throwing is not the right mechanism for a tool signalling its own failure.

## Decision

**1. Deterministic tool outcome signal.** Add an opt-in error result to
`ToolResponse`:

```php
return ToolResponse::error('No record matched the query.');
// $response->isError() === true
```

The library determines each tool's outcome deterministically rather than by
string-sniffing:

- Tool not registered → failure, `error_type = not_found`
- `execute()` threw a `Throwable` → failure, `error_type = exception`
  (caught and guarded so the batch continues)
- Returned `ToolResponse::error()` → failure, `error_type = tool_error`
- Otherwise → success

**2. Distinct `TOOL_FAILED` status plus in-context `ok` flag.** On failure the
library emits `Status::TOOL_FAILED`; on success it emits `TOOL_COMPLETED`. Both
carry `ok` (bool) and `tool_call_id`. `TOOL_FAILED` additionally carries `error`,
`error_type`, and (for thrown failures) `exception_class`. Consumers that switch
on status use `TOOL_FAILED`; consumers watching a single completion event use
`ok`. `error_type` is a plain string (values exposed as `Status::TOOL_ERROR_*`
constants) so it serialises cleanly for every emitter.

**3. `tool_call_id` on all lifecycle events** so consumers can correlate
calling/executing/completed for the same call — required when one batch calls the
same tool multiple times.

The change is additive and backward-compatible: existing tools returning plain
strings or non-error `ToolResponse` values are treated as successes, and the
existing `TOOL_COMPLETED` template is unchanged.

## Alternatives Considered

**Throw exceptions from tools to signal failure (per ADR-0030):** Rejected for
this domain. A tool failure must be reported back to the model as tool output so
it can adapt; an uncaught exception aborts the turn instead. The library still
*catches* exceptions from `execute()` and maps them to a tool-level failure — so
exceptions remain valid, they are just not the required signalling path.

**String-sniffing the output for an `{"error": ...}` shape:** Rejected. There is
no agreed error schema, so detection would be unreliable — the exact problem this
ADR removes.

**`ok` in context only (no `TOOL_FAILED`):** Rejected. Status-switching consumers
and the per-status `StatusMessageFormatter` want a distinct status to react to.
Both are provided.

**A backed `enum` for `error_type`:** Rejected. The value must be flattened to a
string to travel in the `array` status context (JSON for SSE, raw array for
callbacks, interpolation for the formatter), so the enum would never reach
consumers. String constants match the existing `Status` class style.

## Consequences

- Consumers can render tool success vs failure per call, across every emitter.
- One failing tool no longer aborts the batch; remaining tools run.
- Tools gain an opt-in, deterministic failure signal (`ToolResponse::error()`)
  without being forced to change — plain-string returns still work.
- A second, intentional error-signalling path now exists alongside ADR-0030's
  exceptions, scoped specifically to tool outputs. This is documented here to
  avoid the appearance of contradicting ADR-0030.
- `error_type` values become part of the public status contract (the
  `Status::TOOL_ERROR_*` constants); adding a new failure category is additive,
  changing an existing value would be a breaking change.
