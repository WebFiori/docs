# ADR-0037: AI: ModelRouter — Intelligent Multi-Provider Routing

**Date:** 2026-08-16
**Updated:** 2026-08-23
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

### Integration with ModelAliases

`ModelRouter` works with **logical tier names** (e.g., `'fast'`, `'smart'`,
`'coding'`) rather than provider-specific model IDs. Each provider in the
router can carry a `ModelAliases` registry that resolves the tier name to its
own model ID. This cleanly separates two concerns:

- **Router strategies** work in terms of logical tiers — portable across providers
- **ModelAliases** handle provider-specific naming — updated in one place

```
Request
   │
   ▼
ModelRouter  ← picks tier based on request characteristics
   │              ('fast', 'smart', 'coding', etc.)
   ▼
Provider (with ModelAliases)  ← resolves tier → actual model ID
   │              'fast' → 'gpt-4o-mini' (OpenAI)
   │              'fast' → 'gemini-2.5-flash' (Google)
   ▼
API Call
```

```php
$aliases = new ModelAliases([
    'fast'  => ['openai' => 'gpt-4o-mini',  'google' => 'gemini-2.5-flash'],
    'smart' => ['openai' => 'gpt-4o',        'google' => 'gemini-2.5-pro'],
]);

$openai = new OpenAIClient(...);  $openai->setModelAliases($aliases);
$google = new GoogleClient(...);  $google->setModelAliases($aliases);

// Providers keyed by tier name — same names as the aliases
$router = new ModelRouter([
    'fast'  => $openai,   // 'fast' alias → gpt-4o-mini for this provider
    'smart' => $google,   // 'smart' alias → gemini-2.5-pro for this provider
]);

$router->setStrategy(new TaskComplexityStrategy([
    'default' => 'fast',
    'complex' => 'smart',
]));

// Router picks tier, provider resolves to its model ID
$response = $router->chat($messages);
```

Swapping a provider is transparent to routing logic — only the alias table
needs updating when model versions change.

### Routing Priority Stack (highest to lowest)

```
1. force_provider in chat() options   — per-call caller override
2. forceRoute() configuration         — developer locks a route
3. Rule-based routing                 — developer-defined conditions
4. Tool-based routing                 — model decides via tool call
5. Default tier                       — fallback when nothing matched
```

### Transparent Handoff

The routed provider receives the original messages unchanged. The router
injects the resolved tier name as the `model` option so the provider's
`ModelAliases` can resolve it to the correct model ID.

```
User messages
     │
     ▼
ModelRouter classifies task → picks tier ('smart')
     │
     ▼  options['model'] = 'smart'
Google provider (with aliases) → resolves 'smart' → 'gemini-2.5-pro'
     │
     ▼
Response returned to caller
```

### Three Routing Modes

```php
RoutingMode::RULE    // developer-defined conditions only, no LLM call
RoutingMode::TOOL    // model decides via tool call
RoutingMode::HYBRID  // rules first, tool-based for unmatched (default)
```

### API

```php
$router = new ModelRouter(
    providers: [
        'fast'    => $openaiClient,   // tier → provider
        'smart'   => $googleClient,
        'coding'  => $claudeClient,
    ],
    default: 'fast',
);

// Rule-based: explicit condition → tier name
$router->addRule(
    condition: fn(array $messages) => $this->hasImageRequest($messages),
    tier: 'imaging',
    priority: 10,
);

// Force a specific tier globally
$router->forceRoute('coding');

// Observability callback
$router->onRoute(function (string $tier, ProviderInterface $provider, string $reason): void {
    // Log which tier was chosen and why
});

// Per-call override
$response = $router->chat($messages, [
    'force_provider' => 'smart',  // tier name or ProviderInterface instance
]);

// Normal call — router decides
$response = $router->chat($messages);
```

### Built-in Strategies

| Strategy | Logic |
|----------|-------|
| `AlwaysStrategy` | Always routes to a specific tier |
| `TokenLengthStrategy` | Short → fast tier, long → smart tier |
| `KeywordStrategy` | Pattern matching on message content |
| `TaskComplexityStrategy` | Combines token length + tool count + keyword signals |
| `CascadeStrategy` | Try fast tier first, retry with smart tier if response is low quality |

### TaskComplexityStrategy signals

- Message length (short vs long)
- Number of tools available
- Keywords indicating complexity: "compare", "analyze", "summarize", "generate report"
- File attachments present
- Conversation length

## Alternatives Considered

**Using raw model names as router keys (original design):**
`$router->addRoute('coding', $claudeClient, 'description')` tied routing
to the provider instance directly. Rejected in favour of tier names + aliases
because:
- Tier names are portable — `'smart'` means the same thing regardless of provider
- Alias tables let you swap model versions without touching routing logic
- Routing strategies can be written once and reused across provider configurations

**Relay-based routing:**
The default model classifies, calls a tool, receives the specialist's response,
and relays it to the user. Rejected because:
- Two LLM calls and double token cost on every routed request
- Default model may paraphrase or alter the specialist's output

**Separate router class (not implementing ProviderInterface):**
Requires changes at every call site and cannot be used as a drop-in replacement.

**Config-only routing (no tool-based):**
Only rule-based routing. Simpler but brittle — keyword rules miss intent and
require constant maintenance.

## Consequences

**Easier:**
- Strategy logic is written in terms of logical tiers — no provider-specific knowledge needed
- Swapping providers or updating model versions requires only alias table changes
- `ModelRouter` composes with itself — nested routers for multi-tier specialization
- Existing code that accepts `ProviderInterface` works with `ModelRouter` unchanged
- Hybrid mode gives zero-overhead routing for known patterns with intelligent fallback

**Harder:**
- Tool-based routing adds one classification API call per unmatched request
- Developers must write clear tier descriptions for the model to classify correctly
- Streaming with tool-based routing requires the classification call to complete
  before the stream can start — adds latency to the first token
