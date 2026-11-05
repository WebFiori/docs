# ADR-0059: AI: Final Synthesis Pass When the Tool-Call Loop Hits Its Iteration Cap

**Date:** 2026-11-05
**Status:** Accepted

## Context

The auto-execute tool loop in `AbstractClient::chat()` runs while the model
keeps requesting tools, bounded by `ChatOption::MAX_TOOL_ITERATIONS` (default
10):

```php
while ($response->hasToolCalls() && $iteration < $maxIterations) {
    // append the assistant tool-call message + tool results, build the next
    // request, send it, parse $response
}
```

When the loop exits because `$iteration` reached the cap **while the model is
still requesting tool calls**, it simply falls through and returns that last
response. The returned `ChatResponse` therefore carries *tool calls*, not prose
— from the caller's perspective, an empty answer. The model never gets a turn
to synthesize a text answer from the work already done.

The failure is most visible on long tool chains: a turn that would have answered
in one more step instead returns nothing usable once it hits the cap.
Applications work around this with their own post-loop "empty answer" retry,
which wastes the iterations already burned and adds a round-trip after the fact.
That retry lives in application/orchestrator code, not in this library.

No existing ADR specifies the loop's termination behavior. ADR-0056 governs
re-applying the context-window strategy *between* iterations, and ADR-0054
governs per-tool outcome reporting, but neither addresses what happens when the
iteration budget is exhausted. This is therefore a new decision.

## Decision

Add an **opt-in final synthesis pass**. When the loop exits **because the
iteration cap was hit** *and* tool calls are still pending, issue exactly one
more request with tools removed, forcing the model to answer in text. The
feature is off by default and fully backward compatible.

### 1. Two typed `ChatOption` constants (additive)

- `ChatOption::FINAL_SYNTHESIS_ON_MAX_ITERATIONS` — `bool`, default `false`.
  Enables the behavior.
- `ChatOption::FINAL_SYNTHESIS_PROMPT` — `?string`, default `null`. When set, it
  is injected as a synthesis instruction on the final request; when `null`, no
  prompt is injected (tools are simply removed and the conversation resent).

### 2. Trigger — cap-exhaustion, not a fixed iteration

After the `while` loop:

```php
if ($finalSynthesis && $iteration >= $maxIterations && $response->hasToolCalls()) {
    // build one tools-off request and send it; replace $response with the result
}
```

It fires **only** when the cap was the reason for exit *and* tool calls are
still pending. A run that converged normally (the model stopped calling tools)
is untouched. This is deliberately **not** "strip tools at iteration `max-1`":
the model may converge well before the cap, and stripping tools early would
sabotage runs that were going to finish on their own.

### 3. Removing tools via modified `$options` (no provider-specific code)

Every provider reads tools from `$options[ChatOption::TOOLS]` and builds its
request through `buildChatRequest($messages, $options)` (via
`buildIncrementalChatRequest()`). The synthesis request is therefore built with
a **copy of `$options`** that has `ChatOption::TOOLS` unset and
`ChatOption::AUTO_EXECUTE_TOOLS` off. No provider-specific branching and no new
abstract method — every current and future provider inherits the behavior.

### 4. Dangling tool calls — send up to the last completed tool round (Option A)

The last assistant message requests tool calls that were never executed. Most
provider APIs reject a follow-up containing assistant tool-call messages with no
matching tool results. The synthesis request therefore sends the conversation
**up to the last completed tool round** — the unanswered tool-call message is
dropped — then asks the model to answer. A cross-provider guard test asserts the
resulting request is wire-valid on every provider.

### 5. Non-streaming now; streaming via ADR-0060

The non-streaming `chat()` path ships first and is the implemented scope of this
ADR. The streaming path was initially intended for the same version, but a code
audit showed `streamChat()` has **no auto-execute tool loop** and therefore no
iteration cap to trigger synthesis from — and that streamed tool calls are not
even surfaced on some providers (OpenAI, Google generateContent). The streaming
final-synthesis pass is therefore **deferred to and delivered with ADR-0060
(Streaming Auto-Execute Tool Loop)**: once that loop exists, this ADR's trigger
(cap hit + pending tool calls → one tools-off request) applies to it, with the
synthesized tokens flowing through `onToken` / `onComplete`.

### 6. Cross-cutting behavior

- The cancellation token (ADR-0057) rides the synthesis request (it already does
  via `buildIncrementalChatRequest()` → `withCancellationToken()`), so the extra
  round-trip is cancellable.
- The context-window strategy (ADR-0056) must not double-apply on the synthesis
  request.
- A status event, a log line, and a metric mark the synthesis pass so callers
  can see the extra round-trip.

## Alternatives Considered

- **Strip tools at iteration `max-1` — rejected.** Uses a fixed iteration as the
  trigger, which sabotages runs that would converge earlier. Cap-exhaustion is
  the correct trigger.
- **Append synthetic "not executed — budget reached" tool results (Option B) —
  deferred.** Preserves the model's intent to call more tools, but is more
  complex and more provider-variable than dropping the unanswered message
  (Option A). Revisit only if a provider needs it.
- **A post-hoc empty-answer retry — rejected for the library.** This is what
  applications do today: it wastes the iterations already spent and adds a late
  round-trip. The in-loop synthesis pass avoids both. Any such retry remains an
  application-level concern, explicitly out of scope here.
- **Always-on (no opt-in) — rejected.** The extra round-trip has a cost and
  changes output semantics; it must be opt-in and default off for backward
  compatibility.

## Consequences

- Callers can get a usable text answer when a turn exhausts its tool budget,
  with one bounded extra request instead of an after-the-fact retry.
- Fully additive and backward compatible: two options defaulting to `false` /
  `null`; existing callers are unaffected.
- Dropping the dangling tool-call message (Option A) means the model's final
  intended-but-unexecuted tool calls are not represented in the synthesis
  request; the model answers from the completed rounds only.
- One more provider-agnostic terminal behavior lives in `AbstractClient`, so all
  providers inherit it; the cross-provider guard test must keep the tools-off
  request wire-valid as providers evolve.

## Tracking

WebFiori/ai#189.
