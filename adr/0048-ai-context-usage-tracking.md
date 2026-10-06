# ADR-0048: AI: Context Window Usage Tracking

**Date:** 2026-08-30
**Status:** Proposed

## Context

Applications built on the library need to know how much of a model's context
window a request will consume — both to warn users before sending ("you've used
6k of 8k tokens") and to drive UI gauges. Today the pieces exist but are
disconnected:

- `AbstractClient::countTokens($messages, $tools)` returns an estimated token
  count (character-ratio via `TokenEstimator`, ~5–10% margin).
- `AbstractClient::getRemainingTokens($messages, $tools)` returns remaining
  budget, but **only if a `ContextWindowStrategy` is set** (that is where the
  `maxTokens` ceiling lives) and the strategy exposes `getMaxTokens()`.
- The exact prompt token count from a provider is available only *after* a call
  via `$response->getUsage()->getPromptTokens()`.

There is no single call that returns a structured snapshot (used / max /
remaining / percentage), no way to query usage without configuring a truncation
strategy, and no built-in knowledge of per-model context window sizes.

## Decision

Add a `ContextUsage` value object and a `getContextUsage()` method on
`AbstractClient` that returns a structured usage snapshot. Add a
`ContextWindowConfig` model→context-window table (mirroring the existing
`PricingConfig` pattern) so the ceiling can be inferred from the model when no
explicit value or strategy is provided.

### `ContextUsage` DTO

Immutable value object, consistent with `CostResult` / `RateLimitStatus`:

```php
$usage = $client->getContextUsage($messages, $tools);

$usage->getUsedTokens();       // int    — consumed (estimated or actual)
$usage->getMaxTokens();        // ?int   — context window ceiling (null if unknown)
$usage->getReservedTokens();   // int    — reserved for completion (0 if no strategy)
$usage->getAvailableTokens();  // ?int   — max - reserved (null if max unknown)
$usage->getRemainingTokens();  // ?int   — max(0, available - used) (null if max unknown)
$usage->getUsedPercentage();   // ?float — used / max * 100 (null if max unknown)
$usage->isOverBudget();        // bool   — used > (max - reserved); false if max unknown
$usage->isEstimated();         // bool   — true if from estimation, false if from response usage
```

Percentage is computed **against `max`**, not against available budget — a UI
gauge reads more intuitively as "77% of the window full". The
reserved-for-completion concern is surfaced separately via `isOverBudget()`.

### `getContextUsage()` signature

```php
public function getContextUsage(
    array $messages,
    array $tools = [],
    ?int $maxTokens = null,
    ?ChatResponse $response = null,
): ContextUsage;
```

### Ceiling resolution order

The `maxTokens` ceiling is resolved from the first available source:

1. Explicit `$maxTokens` argument (option B — always wins)
2. Context window strategy's `getMaxTokens()` if a strategy is set
3. `ContextWindowConfig` lookup by the client's current model (option C)
4. `null` — usage still reports `usedTokens`; `max`/`remaining`/`percentage`
   are `null` and `isOverBudget()` is `false`

Unknown ceiling degrades gracefully — no crash, no guessed number.

### Estimated vs actual

Both are supported because they answer different questions:

- **Estimated** (default, from input messages): "before I send, will this fit?"
  — the pre-send warning use case.
- **Actual** (when a `ChatResponse` with usage is passed): "what did the last
  turn actually consume?" — uses the provider's exact `promptTokens`.

```php
// Pre-send: estimated
$usage = $client->getContextUsage($messages, $tools);
$usage->isEstimated(); // true

// Post-response: exact
$usage = $client->getContextUsage($messages, $tools, response: $lastResponse);
$usage->isEstimated(); // false
```

When a response is provided **and** carries usage data, `usedTokens` uses the
exact `promptTokens`; otherwise it falls back to estimation. `isEstimated()`
lets the UI render "~6.2k" versus "6,200" honestly. The two are not
interchangeable: a response reflects what a *previous* request sent, so newly
added, unsent messages still require estimation.

### `ContextWindowConfig`

Mirrors `PricingConfig` exactly — a data table with built-in defaults, fully
overridable, zero dependencies:

```php
use WebFiori\Ai\Context\ContextWindowConfig;

$config = new ContextWindowConfig();
$config->getContextWindow('gpt-4o');                   // 128000
$config->getContextWindow('gemini-2.5-flash');         // 1048576
$config->getContextWindow('claude-sonnet-4-20250514'); // 200000
$config->getContextWindow('unknown-model');            // null

$config->setContextWindow('my-fine-tune', 32000);      // override / add

$client->setContextWindowConfig($config);              // sensible default auto-created
```

- Ships with a default table of known models.
- Unknown model → `null`.
- Like the pricing table, sizes drift as providers ship models; the table needs
  periodic updates and can be stale. Documented, and overridable by users.

## Alternatives Considered

**Require a context window strategy (status quo):**
`getRemainingTokens()` already works this way. Rejected as the sole option —
forces users to configure truncation just to *read* usage. The explicit
`$maxTokens` param and the config table remove that coupling.

**Percentage against available budget (max − reserved):**
Can exceed 100%, which is useful for "over input budget" signalling but
confusing for a gauge. Rejected in favour of percentage-against-max plus a
separate `isOverBudget()` flag.

**Estimation only (no actual):**
Simpler, but throws away the exact `promptTokens` the provider already returns.
Rejected — post-response accuracy is cheap to support and valuable.

**Actual only (read `$response->getUsage()`):**
Already possible, but doesn't help the pre-send case (no response yet).
Rejected as insufficient on its own.

**Auto-guess context window from model name heuristics:**
Fragile and dishonest. Rejected in favour of an explicit, overridable table
that returns `null` for unknowns.

## Consequences

**Easier:**
- One call returns a complete, structured usage snapshot for UI gauges and
  pre-send warnings.
- Usage can be queried without configuring a truncation strategy (explicit
  `maxTokens` or the model table).
- Exact post-response accounting when a response is available, estimated
  otherwise, with an honest `isEstimated()` flag.
- `ContextWindowConfig` follows the familiar `PricingConfig` pattern.

**Harder:**
- The model context-window table needs maintenance and will occasionally be
  stale (same tradeoff as `PricingConfig`).
- Two sources of "used tokens" (estimated vs actual) require callers to
  understand they answer different questions; the `isEstimated()` flag mitigates
  this.
- Estimation retains the `TokenEstimator` ~5–10% margin; not a substitute for
  provider-reported counts.
