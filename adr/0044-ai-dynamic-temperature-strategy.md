# ADR-0044: AI: Dynamic Temperature Strategy

**Date:** 2026-08-23
**Status:** Accepted

## Context

Temperature controls the creativity/randomness of AI responses. Developers
currently set it manually per-call or leave it at the provider default. This
is fine for single-purpose applications but brittle for general-purpose assistants
that handle diverse request types — a factual lookup should get 0.3 while a
creative writing request should get 0.9.

A `TemperatureStrategyInterface` allows the temperature to be set automatically
based on the characteristics of the request.

## Decision

Add `TemperatureStrategyInterface` evaluated in `AbstractClient::chat()` before
building the request. Temperature is only applied when not already explicitly
set in `options['temperature']`.

### Design Decisions

**1. Placement: setTemperatureStrategy() on AbstractClient**

```php
$client->setTemperatureStrategy(new TaskBasedTemperatureStrategy());
$response = $client->chat($messages); // temperature auto-set
```

**2. Only applies when temperature not explicitly set**

If the caller passes `options['temperature']`, the strategy is skipped. The
strategy is a default, not a mandate.

```php
// Strategy applies:
$client->chat($messages);

// Strategy skipped — caller knows what they want:
$client->chat($messages, ['temperature' => 0.2]);
```

**3. ChatContext for extensibility**

The strategy receives a `ChatContext` value object rather than raw
`(array $messages, array $options)`. This future-proofs the interface — new
signals (provider name, model, previous usage, etc.) can be added without
breaking existing implementations.

```php
class ChatContext {
    public function __construct(
        public readonly array $messages,
        public readonly array $options,
    ) {}
}

interface TemperatureStrategyInterface {
    public function temperature(ChatContext $context): float;
}
```

**4. Two built-in strategies**

`FixedTemperatureStrategy` — always returns the same temperature. Useful for
testing, baseline comparison, or simple applications.

```php
$client->setTemperatureStrategy(new FixedTemperatureStrategy(0.5));
```

`TaskBasedTemperatureStrategy` — classifies the request by scanning user message
content for keywords and structural signals. Configurable keyword buckets with
sensible defaults.

```php
// Default buckets:
$client->setTemperatureStrategy(new TaskBasedTemperatureStrategy());

// Custom buckets:
$client->setTemperatureStrategy(new TaskBasedTemperatureStrategy(
    buckets: [
        0.2 => ['sql', 'query', 'select', 'insert'],
        0.7 => ['analyze', 'explain', 'compare'],
        0.9 => ['write', 'create', 'draft', 'story'],
    ],
    default: 0.7
));
```

**Default keyword buckets:**

| Temperature | Keywords | Task type |
|-------------|----------|-----------|
| 0.3 | lookup, define, what is, when was, who is, list, calculate, convert | Simple data lookup |
| 0.7 | analyze, analyse, compare, explain, summarize, summarise, describe, evaluate | Analytical / multi-tool |
| 0.9 | write, generate, create, draft, story, poem, essay, brainstorm, imagine | Creative / document generation |
| 0.7 *(default)* | — | Balanced fallback |

Structural signals also apply:
- `json_mode: true` or `json_schema` present → cap at 0.3 (structured output needs precision)
- Tools present → cap at 0.7 (agentic tasks need balanced reasoning)

## Alternatives Considered

**Plain (array $messages, array $options) signature:**
Adding any new signal (provider name, model, usage data) would require breaking
all existing implementations. `ChatContext` provides a stable extension point.
Rejected.

**Always apply (override explicit temperature):**
Violates the principle of least surprise — if the developer explicitly sets
temperature, they know what they want. Strategy should be a default, not a
mandate. Rejected.

**Classifier-based (LLM decides temperature):**
Would require an extra API call to classify the task type. For something as
lightweight as temperature selection, keyword matching is sufficient.
Rejected.

**Keywords only (no structural signals):**
`json_mode: true` is a strong signal for low temperature — a model generating
structured JSON should be deterministic. Ignoring options misses this.
Rejected.

## Consequences

**Easier:**
- Temperature tuning is automatic for diverse request types
- No per-call temperature management for general-purpose assistants
- `ChatContext` is reusable for future request-level strategies
- Existing code unchanged — strategy is opt-in

**Harder:**
- Keyword matching can misclassify edge cases
- Default buckets may not fit all domains — requires customization
