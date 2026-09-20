# ADR-0048: AI: Domain Boundaries — Guardrails for Off-Topic Questions

**Date:** 2026-09-20
**Status:** Accepted

## Context

When agents are deployed in production, users sometimes ask questions that are
completely outside the agent's domain — asking a "Business AI" about the
weather, song lyrics, or personal advice. Left unchecked, the model will
attempt to answer these off-topic questions, which:

1. Wastes tokens (and money) on responses the agent was never meant to give.
2. Produces unreliable answers outside the agent's area of expertise.
3. Misses the opportunity to redirect the user back to what the agent can
   actually help with.

An `AgentProfile` already *describes a domain*: its `identity`, `skills`, and
`constraints` collectively define what the agent is for. The natural question is
whether we can turn that domain description into an enforceable boundary that
short-circuits off-topic requests **before** the provider is ever called.

Two placement questions had to be answered:

1. **Where does the boundary logic live?** It must be reusable, not duplicated.
2. **Who enforces it?** The profile (per-agent, travels with the agent) or the
   chat call (universal, but the core client has no concept of a domain today)?

## Decision

Introduce a single, provider-agnostic value object —
`WebFiori\Ai\Tool\DomainBoundaries` — as the boundary **engine**, and wire it in
at **two entry points**: the profile (primary) and the chat call (optional).

### Design Decisions

**1. `DomainBoundaries` value object — one engine, reused everywhere**

The decision logic (is this question in-domain?) is identical regardless of
where it runs. It lives in one class with a `check(string $question): ?string`
method that returns a redirect message when the question is out of scope, or
`null` to proceed to the model.

```php
use WebFiori\Ai\Tool\DomainBoundaries;

$boundaries = new DomainBoundaries(
    inScope: ['Sales data', 'Stock levels', 'Forecasts'],
    adjacentAllowed: ['Market factors affecting business metrics'],
    blockPatterns: ['weather|forecast|temperature', 'lyrics|song|movie'],
    redirectTemplate: "That's outside my focus as Business AI! I specialize in "
        . "sales, stock, and forecast data. Want me to check inventory levels "
        . "or analyze revenue trends instead?",
);

$redirect = $boundaries->check('What is the weather tomorrow?');
// → returns the redirect template (blocked)

$redirect = $boundaries->check('How do market factors affect our stock?');
// → returns null (adjacent-allowed wins over the block pattern)
```

**2. Block-list with adjacency override (v1 default: `regex` strategy)**

The default strategy is fast, deterministic regex matching. A question that
matches a `block_pattern` is redirected — **unless** it also matches an
`adjacent_allowed` keyword, in which case it is let through to the model.
Adjacency wins so that tangential-but-relevant questions ("how do geopolitical
factors affect our stock levels") are not falsely blocked. Matching is
case-insensitive, and a malformed pattern is skipped safely rather than raising
a warning, so a bad profile can never break execution.

**3. Profile is the primary owner (`AgentProfile::getDomainBoundaries()`)**

Because the profile already declares the domain, the boundary belongs alongside
the other profile fields. `AgentProfile` gains an optional, nullable
`domain_boundaries` field threaded through the constructor, `getDomainBoundaries()`,
`fromArray()`, `toArray()`, and the inheritance merge — so it loads from JSON/URL,
round-trips, and inherits like every other field. It is **not** injected into the
rendered system prompt: enforcement is programmatic precisely because prompt-only
guardrails can be argued away by the model.

Enforcement happens in `AgentTool::execute()`, before any messages are built or
the provider is called:

```php
public function execute(array $arguments): string|ToolResponse {
    $task = $arguments['task'];
    $boundaries = $this->profile->getDomainBoundaries();

    if ($boundaries !== null) {
        $redirect = $boundaries->check($task);
        if ($redirect !== null) {
            return $redirect; // soft refusal — zero provider calls
        }
    }
    // ... existing flow unchanged
}
```

Example JSON profile:

```json
{
    "identity": "You are Business AI.",
    "skills": ["Sales data", "Stock levels", "Forecasts"],
    "domain_boundaries": {
        "strategy": "regex",
        "in_scope": ["Sales data", "Stock levels", "Forecasts"],
        "adjacent_allowed": ["Market factors affecting business metrics"],
        "block_patterns": ["weather|forecast|temperature", "lyrics|song|movie"],
        "redirect_template": "That's outside my focus as Business AI! ..."
    }
}
```

**4. Chat is an optional, opt-in entry point (`ChatOption::DOMAIN_BOUNDARIES`)**

Profiles only cover flows that go through an `AgentTool`. To also protect raw
`$client->chat($messages)` calls that have no profile, a `DomainBoundaries`
instance may be passed per request via `ChatOption::DOMAIN_BOUNDARIES`. When
present, `AbstractClient::chat()` checks the latest user message and, on a
redirect, returns a synthetic `ChatResponse` (assistant message = redirect,
`finishReason = 'domain_boundary'`, zero-token `Usage`) **without** an HTTP call.

```php
$client->chat($messages, [
    ChatOption::DOMAIN_BOUNDARIES => $boundaries,
]);
```

This keeps the guardrail out of the core code path by default — existing chat
behavior is unchanged unless the option is explicitly supplied.

**5. Backward compatible — opt-in at both layers**

Profiles without `domain_boundaries` behave exactly as before, and `chat()`
without the option is untouched. The feature is additive at every layer.

## Alternatives Considered

**Profile instructions only (prompt-level).** Add boundary rules to the system
prompt and rely on the model to refuse. Works partially but is bypassable —
models can be persuaded to ignore instructions — and it still costs a full
provider round-trip. Rejected as the sole mechanism; programmatic enforcement is
used instead.

**Embedding-similarity scope check as the v1 default.** Embed the domain and the
question, redirect below a similarity threshold. More accurate for novel
phrasings, and the library already has the infrastructure (`RagProviderInterface`,
`embed()`). But it adds latency and cost to every request. Deferred: the design
reserves a `semantic` strategy for this, but v1 defaults to regex for speed and
determinism.

**Mini-LLM intent classifier.** Most accurate, but adds a second model call,
cost, and operational complexity. Rejected for v1.

**Chat-only enforcement (no profile field).** Universal, but pushes a
domain concept into `AbstractClient`, which today knows nothing about profiles or
domains, and forces a "which message do we check?" heuristic into the core
contract. Rejected as the primary mechanism; kept as an optional entry point.

**Profile-only enforcement (no chat option).** Clean, but leaves profile-less
raw `chat()` calls unguarded. Rejected as the sole mechanism; the chat option
fills the gap without changing default behavior.

## Consequences

**Easier:**
- Off-topic questions are caught before the provider call, saving tokens and cost.
- The domain boundary travels with the profile (JSON, URL, inheritance, versioning).
- Soft redirects steer users back to what the agent can do, improving UX.
- One engine, two entry points — no duplicated logic; raw chat can opt in.
- Boundary responses are observable (`finishReason = 'domain_boundary'`, zero usage).

**Harder:**
- Regex block-lists require authoring and tuning; overly broad patterns can
  false-positive (mitigated by `adjacent_allowed`).
- The `domain_boundaries` field must be maintained across all `AgentProfile`
  serialization/inheritance paths.
- The optional `semantic` strategy, when enabled, reintroduces per-request
  embedding latency and cost (a deliberate, opt-in trade-off).
