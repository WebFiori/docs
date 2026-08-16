# ADR-0037: AI: ModelRouter — Intelligent Multi-Provider Routing

**Date:** 2026-08-16
**Status:** Accepted

## Context

Applications often need different AI models for different tasks. Claude excels
at code, Gemini Pro at reasoning, DALL-E at images. Currently, a developer
must manually decide which provider to use per request and wire the routing
logic themselves. There is no built-in mechanism to route requests to the most
appropriate model based on task type.

Three related needs exist:

1. **Intelligent routing** — route to the best provider for the task
2. **Provider enforcement** — force a specific provider regardless of routing
3. **Observability** — know which provider was chosen and why

## Decision

Add a `ModelRouter` class that implements `ProviderInterface`. It is a
drop-in replacement anywhere a provider is used, including inside other routers
(nested routing for multi-tier specialization).

**Routing Priority Stack (highest to lowest):**

```
1. force_provider in chat() options   — per-call caller override
2. forceRoute() configuration         — developer locks a route
3. Rule-based routing                 — developer-defined conditions
4. Tool-based routing                 — model decides via tool call
5. Fallback provider                  — default when nothing matched
```

**Transparent Handoff:**

The routed provider receives the original messages unchanged, not a relay
from the default model. This is critical for code generation — the specialist
model should answer directly, not have its output paraphrased by the router.

```
User messages
     │
     ▼
ModelRouter classifies task
     │
     ▼ (library intercepts, swaps provider transparently)
Specialist provider receives original messages
     │
     ▼
Response returned to caller
```

**Three Routing Modes:**

```php
RoutingMode::RULE    // developer-defined conditions only, no LLM call
RoutingMode::TOOL    // model decides via tool call
RoutingMode::HYBRID  // rules first, tool-based for unmatched (default)
```

Hybrid is the default — rules handle obvious cases with zero overhead,
tool-based handles ambiguous ones.

**API:**

```php
$router = new ModelRouter($defaultClient);

// Register routes
$router->addRoute('coding',    $claudeClient,  'Code writing, debugging, review');
$router->addRoute('reasoning', $geminiPro,     'Math, logic, multi-step analysis');

// Rule-based override (no LLM call needed)
$router->addRule(
    condition: fn(array $messages) => $this->hasImageRequest($messages),
    provider: $dalleClient,
    priority: 10,
);

// Force a specific provider globally
$router->forceProvider($geminiFlashClient);

// Force a specific route to a specific provider
$router->forceRoute('coding', $geminiFlashClient);

// Observability
$router->onRoute(function (string $route, ProviderInterface $provider): void {
    // Log which model was chosen and why
});

// Per-call override (highest priority)
$response = $router->chat($messages, [
    'force_provider' => $geminiFlashClient,
]);

// Normal call — router decides
$response = $router->chat($messages);
```

**Tool hiding:** When `force_provider` is set (either globally or per-call),
the routing tool definition is not sent to the model. Offering a tool the
system will not honour is misleading.

**Implements `ProviderInterface`:** All `chat()`, `streamChat()`, `embed()`,
`generateImage()`, and `healthCheck()` calls are forwarded to the resolved
provider. Non-chat operations (embed, image) use `forceProvider` if set,
otherwise the fallback provider.

## Alternatives Considered

**Relay-based routing (default model relays to specialist):**
The default model classifies, calls a tool, receives the specialist's response,
and relays it to the user. Rejected because:
- Two LLM calls and double token cost on every routed request
- Default model may paraphrase or alter the specialist's output
- Code, structured output, and citations are particularly vulnerable to relay distortion

**Separate router class (not implementing ProviderInterface):**
A `Router::route($messages)` that returns a provider, leaving the caller to
call `chat()`. Rejected because it requires changes at every call site and
cannot be used as a drop-in replacement.

**Config-only routing (no tool-based):**
Only rule-based routing, no LLM classification. Simpler but brittle — keyword
rules miss intent and require constant maintenance as use cases grow.

## Consequences

**Easier:**
- Multi-model applications require no routing logic at the call site
- `ModelRouter` composes with itself — nested routers for multi-tier specialization
- Existing code that accepts `ProviderInterface` works with `ModelRouter` without changes
- Hybrid mode gives zero-overhead routing for known patterns with intelligent fallback

**Harder:**
- Tool-based routing adds one classification API call per unmatched request
- Developers must write clear route descriptions for the model to classify correctly
- Streaming with tool-based routing requires the classification call to complete
  before the stream can start — adds latency to the first token
