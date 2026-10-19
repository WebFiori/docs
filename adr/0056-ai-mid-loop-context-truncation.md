# ADR-0056: AI: Re-apply Context Window Strategy Between Tool Iterations

**Date:** 2026-10-19
**Status:** Proposed

## Context

The context window strategy (`ContextWindowStrategyInterface`, e.g.
`SlidingWindowStrategy`, `SummarizingWindowStrategy`) is applied **once per
`chat()` call** — before the tool-execution loop. In
`AbstractClient::chat()`, `applyContextStrategy()` runs at the start, but the
auto-execute tool loop then appends each assistant tool-call message and every
tool result (`$messages[] = ...`) across iterations **without ever re-applying
the strategy**.

As a result, a single turn that chains several tool calls returning large
outputs can grow the prompt far beyond the strategy's configured `maxTokens`.
The accumulated context is re-sent on every subsequent round-trip, so tokens,
latency, and cost compound within one turn.

Observed in production (Google/Vertex, `gemini-3.8-flash`): a document-audit
turn chaining `list_*` + several content-extraction tool calls reached a
**~514K-token prompt** and ~119s latency, despite the provider being configured
with `SlidingWindowStrategy(maxTokens: 150000)`. Each individual tool output was
already bounded; the bloat came purely from un-retruncated accumulation inside
the loop. This defeats the entire purpose of configuring a context strategy —
the bound silently does not hold for the exact workload (long tool chains) that
needs it most.

Two further constraints shape the design:

1. **Incremental-formatting bookkeeping.** The loop sends only the incremental
   tail via `array_slice($messages, $formattedCount)` and rebuilds the request
   through `buildIncrementalChatRequest()`. Any mid-loop truncation that removes
   older messages must reconcile `$formattedCount`, otherwise the outgoing
   "new messages" slice is computed incorrectly (wrong, or negative).

2. **Tool-call ↔ tool-result pairing.** The existing `SlidingWindowStrategy`
   drops *individual* oldest messages purely by token budget
   (`doTruncate()` walks newest→oldest keeping whatever fits). Mid-loop, the
   message list contains assistant messages carrying `tool_calls` and the `tool`
   result messages that answer them. Dropping one half of a pair produces a
   provider-invalid request:
   - **OpenAI** rejects an assistant `tool_calls` message not followed by a
     `tool` message for each `tool_call_id`.
   - **Anthropic** requires every `tool_use` block to have a matching
     `tool_result`.
   - **Bedrock (Converse)** validates `toolUse`/`toolResult` correspondence.

   A token-greedy drop can therefore convert a working turn into a hard 400.
   Pre-loop truncation never hits this because, before any tools run, history is
   just system/user/assistant with no dangling tool pairs.

## Decision

Re-apply the context window strategy **inside** the tool-execution loop —
after appending tool results and before building the next request — behind an
**opt-in configuration**, with a **pairing-safe** truncation path and a
**configurable overflow behavior** (drop vs. summarize).

### 1. Opt-in activation via `ChatOption` (default off)

Add `ChatOption::TRUNCATE_BETWEEN_TOOL_ITERATIONS` (default `false`). Existing
consumers' behavior is unchanged; the single pre-loop truncation is preserved.
When no `contextStrategy` is configured, the option is a no-op.

```php
$response = $client->chat($messages, [
    ChatOption::TOOLS => $tools,
    ChatOption::AUTO_EXECUTE_TOOLS => true,
    ChatOption::TRUNCATE_BETWEEN_TOOL_ITERATIONS => true, // NEW, opt-in
]);
```

When enabled and a `contextStrategy` is set, after the tool-result append and
before `buildIncrementalChatRequest()`:

1. Run the pairing-safe truncation over the full accumulated `$messages`.
2. Reset the incremental-formatting state: set `$formattedCount = 0`, rebuild the
   request from the truncated set, then set `$formattedCount = count($messages)`.
3. Emit `Status::TRUNCATING_CONTEXT` so the mid-loop event is observable.

### 2. Pairing-safe group truncation via a standalone helper

Truncation used inside the loop treats a tool-call interaction as an
**atomic group**:

```
GROUP = [assistant message with tool_calls]
      + [every tool result message answering those tool_call_ids]
```

Rules:

- Drop or keep a **whole group** — never a partial group. Guarantees no orphaned
  `tool_calls` and no orphaned `tool` results.
- Drop **oldest groups first** (recency bias): the most recent assistant
  tool-call drives the current iteration and is always retained.
- The system message is always preserved (existing behavior).
- Non-tool messages (plain user/assistant/system) remain individually
  droppable as today.

Pairing lives in a small, independently-testable helper in the `Context`
namespace — **not** in `ContextWindowStrategyInterface`, since pairing only
matters mid-loop:

```php
// WebFiori\Ai\Context\ToolCallGrouping
final class ToolCallGrouping {
    /** Split a flat message list into ordered atomic units
     *  (singletons for non-tool messages, groups for tool-call+results). */
    public static function group(array $messages): array;

    /** Flatten units back to a message list, preserving order. */
    public static function flatten(array $units): array;
}
```

The in-loop truncation composes this helper with the configured strategy's
token budget. `SlidingWindowStrategy` / `SummarizingWindowStrategy` and the
`ContextWindowStrategyInterface` contract are left unchanged for the pre-loop
path.

### 3. Configurable overflow behavior (drop vs. summarize)

Expose the mid-loop overflow behavior explicitly rather than inferring it from
the configured strategy type:

```php
enum ToolLoopOverflowBehavior {
    case DROP_OLDEST;   // hard-drop oldest groups (pairing-safe) — default
    case SUMMARIZE;     // replace dropped groups with a compact summary
}
```

Wired through two new `ChatOption`s:

```php
$client->chat($messages, [
    ChatOption::AUTO_EXECUTE_TOOLS => true,
    ChatOption::TRUNCATE_BETWEEN_TOOL_ITERATIONS => true,
    ChatOption::TOOL_LOOP_OVERFLOW_BEHAVIOR => ToolLoopOverflowBehavior::SUMMARIZE,
    ChatOption::TOOL_LOOP_SUMMARIZER => $gpt4oMini, // a ProviderInterface
]);
```

- **`TOOL_LOOP_OVERFLOW_BEHAVIOR`** — enum, default `DROP_OLDEST`.
- **`TOOL_LOOP_SUMMARIZER`** — `?ProviderInterface`, default `null`. An injected
  summarizer, consistent with ADR-0043's "separate injected provider" pattern.

**`DROP_OLDEST`** removes oldest groups wholesale (pairing-safe); needs no
summarizer.

**`SUMMARIZE`** replaces the dropped groups' content with a compact synthetic
summary (preserving that the calls happened and their gist, pairing intact).
The summarizer is resolved in this order:

1. `TOOL_LOOP_SUMMARIZER` if provided (explicit wins).
2. Else, if `contextStrategy` is a `SummarizingWindowStrategy`, reuse its
   summarizer.
3. Else → **fail fast** at the start of `chat()` with a configuration exception:
   *"TOOL_LOOP_OVERFLOW_BEHAVIOR=SUMMARIZE requires a summarizer (set
   TOOL_LOOP_SUMMARIZER or configure a SummarizingWindowStrategy)."*

Per ADR-0043, the summarizer should generally be a cheaper/faster model and not
the same near-full client (circular risk). This is documented guidance, not
enforced in code (provider-identity checks are brittle).

## Alternatives Considered

**Subclass override of `buildIncrementalChatRequest()` (app-side, no library
change).** Not viable: `applyContextStrategy()`, `executeTools()`, and the
loop's `$formattedCount` bookkeeping are `private` to `AbstractClient::chat()`.
A subclass can mutate the handed request but cannot keep the incremental slice
consistent with truncation, nor enforce pairing. Fragile.

**Bound/summarize tool outputs at the application tool-handler level.** Helps,
but is per-tool and per-app, does not bound cumulative growth across many
individually-bounded outputs, and does not solve the library limitation for
other consumers.

**Lower `MAX_TOOL_ITERATIONS`.** Only caps how far context can grow; a few large
outputs can already exceed the budget, and it reduces legitimate multi-step
capability.

**Make it default-on (not opt-in).** Rejected to preserve backward compatibility
and match the library's explicit-configuration philosophy (ADR-0034). Revisit if
the opt-in proves to be a silent footgun.

**Reuse `SlidingWindowStrategy` as-is inside the loop (token-greedy, no
grouping).** Rejected: `doTruncate()` is pairing-blind and can emit orphaned
tool-calls/results, producing provider 400s in exactly the large-tool-output
scenario this ADR targets.

**Enforce pairing inside every strategy instead of a shared helper.** Rejected:
pairing only matters mid-loop; pushing tool-call semantics into every
`ContextWindowStrategyInterface` implementation widens the contract for all
callers. A composable grouping helper keeps the concern localized.

**Infer summarize-vs-drop from the strategy type (summarize iff
`SummarizingWindowStrategy`).** Rejected: implicit coupling that is easy to
misread, and prevents summarizing mid-loop while using a non-summarizing
pre-loop strategy. An explicit enum + injectable summarizer decouples the two
concerns.

**Silently fall back to `DROP_OLDEST` when `SUMMARIZE` has no summarizer.**
Rejected in favor of fail-fast: silent degradation hides a misconfiguration and
yields worse context than the user asked for, without signal.

## Consequences

**Easier:**
- A configured context strategy now actually bounds a full multi-step turn, not
  just the first request — tokens, latency, and cost stay within budget during
  tool chains.
- Drop-vs-summarize is an explicit, independent choice, decoupled from the
  pre-loop strategy.
- Mid-loop truncation is observable via `Status::TRUNCATING_CONTEXT`.
- Pairing-safe grouping is reusable and independently testable.

**Harder / trade-offs:**
- Opt-in means affected users must discover and enable the flag; the default
  behavior still grows unbounded.
- Group-atomic dropping is coarser than per-message dropping — a single large
  group is removed wholesale (mitigated by `SUMMARIZE`).
- `SUMMARIZE` adds a possible extra summarizer API call per iteration and more
  test surface.
- `$formattedCount` reconciliation adds a subtle loop invariant that must be
  covered by tests.
- Larger config surface: one enum + two `ChatOption`s + summarizer resolution +
  fail-fast validation.

**Migration:** None required. Behavior is unchanged unless
`TRUNCATE_BETWEEN_TOOL_ITERATIONS` is set to `true`.

**Testing (acceptance criteria):**
- Stubbed provider emitting oversized tool outputs: every outgoing request stays
  within the strategy budget across iterations.
- No outgoing request ever contains an orphaned `tool_calls` message or an
  orphaned `tool` result (pairing integrity), for both `DROP_OLDEST` and
  `SUMMARIZE`.
- `$formattedCount` reconciliation: the incremental slice after a mid-loop
  truncation matches the truncated message set.
- `SUMMARIZE` with no resolvable summarizer fails fast with a clear config
  exception before any HTTP call.
- `SUMMARIZE` reuses `SummarizingWindowStrategy`'s summarizer when
  `TOOL_LOOP_SUMMARIZER` is not provided.
- Option defaults: `TRUNCATE_BETWEEN_TOOL_ITERATIONS=false` and
  `TOOL_LOOP_OVERFLOW_BEHAVIOR=DROP_OLDEST` leave current behavior unchanged.
