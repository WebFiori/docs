# ADR-0039: AI: FallbackProvider — Resilient Multi-Provider Failover

**Date:** 2026-08-19
**Status:** Accepted

## Context

Production AI applications cannot depend on a single provider. Providers
experience outages, rate limits, and network failures. When OpenAI is down,
requests should automatically fall back to Anthropic or Google without
requiring application code changes.

Three related needs exist:

1. **Automatic failover** — retry with another provider when one fails
2. **Load distribution** — spread traffic across providers for cost or capacity
3. **Circuit breaking** — stop hammering a failing provider

This is distinct from ModelRouter (ADR-0037), which selects providers based on
task characteristics. FallbackProvider handles resilience — what happens when
the selected provider fails.

## Decision

Add a `FallbackProvider` class that implements `ProviderInterface`. It wraps
multiple providers and handles failover transparently.

**Core Design:**

```php
$provider = new FallbackProvider(
    providers: [$openai, $anthropic, $google],
    config: new FallbackConfig(
        strategy: FallbackStrategy::SEQUENTIAL,
        failoverOn: [ProviderException::class, HttpException::class, RateLimitException::class],
        maxAttempts: 3,
        circuitBreaker: new CircuitBreakerConfig(
            failureThreshold: 5,
            cooldownSeconds: 60,
            successThreshold: 2,
        ),
    ),
);

// Use like any other provider
$response = $provider->chat($messages);

// Check which provider served the request
echo $provider->getLastUsedProvider();
```

**Three Routing Strategies:**

| Strategy | Behavior |
|----------|----------|
| `SEQUENTIAL` | Try providers in order until one succeeds (default) |
| `ROUND_ROBIN` | Rotate through providers, distributing load |
| `WEIGHTED` | Assign percentage weights to each provider |

**Circuit Breaker Pattern:**

The circuit breaker prevents repeated calls to a failing provider:

```
CLOSED (normal) ──[N failures]──► OPEN (skip provider)
                                       │
                              [cooldown expires]
                                       │
                                       ▼
                                  HALF_OPEN (test)
                                       │
                    ┌──[failure]───────┴───[M successes]──┐
                    ▼                                      ▼
                   OPEN                                  CLOSED
```

States:
- **CLOSED** — Normal operation, requests pass through
- **OPEN** — Provider marked unavailable, requests skip it entirely
- **HALF_OPEN** — Testing recovery, limited requests allowed through

**Configurable Failover Conditions:**

Not all exceptions should trigger failover. Authentication errors indicate
bad credentials — retrying with the same provider won't help, but retrying
with a different provider also won't help. By default, failover triggers on:

- `ProviderException` — API errors (5xx, model not found, etc.)
- `HttpException` — Network errors (timeout, DNS failure, etc.)
- `RateLimitException` — 429 responses

`AuthenticationException` does NOT trigger failover by default — it fails fast.

**Implements `ProviderInterface`:**

FallbackProvider is a drop-in replacement for any single provider:

```php
// Before: single provider
$app->setAiProvider($openai);

// After: resilient multi-provider
$app->setAiProvider(new FallbackProvider([$openai, $anthropic]));
```

All methods (`chat`, `streamChat`, `embed`, `generateImage`, `healthCheck`)
delegate to the resolved provider.

**Composition with ModelRouter:**

FallbackProvider and ModelRouter serve different purposes and compose naturally:

```php
// Resilient providers
$fastProviders = new FallbackProvider([$geminiFlash, $gpt4oMini]);
$smartProviders = new FallbackProvider([$claude, $gpt4o]);

// Intelligent routing over resilient backends
$router = new ModelRouter([
    'fast' => $fastProviders,
    'complex' => $smartProviders,
]);
$router->setStrategy(new TaskComplexityStrategy());
```

**Same Provider, Different Models:**

A single provider class can appear multiple times with different configurations:

```php
$provider = new FallbackProvider([
    new GoogleClient(['model' => 'gemini-2.5-flash']),
    new GoogleClient(['model' => 'gemini-2.5-pro']),
    new GoogleClient(['model' => 'gemini-3.0-pro']),
]);
```

Each instance is independent with its own circuit breaker state.

## Alternatives Considered

**Manual try/catch in application code:**
Developers wrap each AI call in try/catch and manually retry with another
provider. Rejected because:
- Boilerplate at every call site
- No circuit breaking — keeps hammering failing providers
- Each developer reinvents the wheel with different retry logic

**External service mesh (Envoy, Istio):**
Use infrastructure-level retry and circuit breaking. Rejected because:
- Overkill for a PHP library
- Requires infrastructure changes
- Cannot make provider-aware decisions (e.g., parse rate limit headers)

**Strategy interface instead of enum:**
Define `FallbackStrategyInterface` with `getNextProvider()` method. Rejected
because:
- The three strategies (sequential, round-robin, weighted) cover all common cases
- An interface adds complexity without clear benefit
- Custom strategies can be added later if needed without breaking changes

**Unified Router/Fallback class:**
Combine ModelRouter and FallbackProvider into one class. Rejected because:
- Single responsibility — routing and resilience are distinct concerns
- Composition is more flexible than a monolithic class
- Easier to test and reason about separately

## Consequences

**Easier:**
- Production resilience with one line change (wrap providers in FallbackProvider)
- Automatic circuit breaking prevents cascading failures
- Metrics callback enables observability without code changes
- Composes with ModelRouter for intelligent + resilient routing

**Harder:**
- `getLastUsedProvider()` returns provider name, not model — same provider with
  different models returns the same name
- Circuit breaker state is in-memory — resets on process restart
- Weighted strategy uses random selection — not deterministic for testing
