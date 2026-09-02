# Model Aliases and Routing

Two features help you manage which model handles a request: model aliases give logical names to provider-specific model IDs, and the model router picks a provider or tier automatically based on the request.

<meta name="description" content="Use model aliases and the model router in WebFiori AI to map logical model names and route requests by complexity or rules.">

## Model Aliases

Hard-coding model IDs across a codebase makes changes painful. `ModelAliases` lets you define logical names once, mapped per provider, and use those names everywhere:

```php
use WebFiori\Ai\ModelAliases;

$aliases = new ModelAliases([
    'fast'  => ['openai' => 'gpt-4o-mini', 'google' => 'gemini-2.5-flash'],
    'smart' => ['openai' => 'gpt-4o',      'google' => 'gemini-2.5-pro'],
]);

$client->setModelAliases($aliases);

$response = $client->chat($messages, ['model' => 'fast']);
// resolves to gpt-4o-mini on OpenAI, or gemini-2.5-flash on Google
```

Each client resolves an alias to the ID for its own provider, so the same `'fast'` request works whichever provider you configured. If a name is not an alias, it is passed through as a literal model ID.

## Model Router

`ModelRouter` chooses among several providers (or tiers) for each request. Construct it with a map of named tiers and an optional default:

```php
use WebFiori\Ai\Routing\ModelRouter;
use WebFiori\Ai\Routing\Strategy\TaskComplexityStrategy;

$router = new ModelRouter(
    ['fast' => $geminiFlash, 'smart' => $geminiPro],
    'fast', // default tier
);

$router->setStrategy(new TaskComplexityStrategy('fast', 'smart'));

$response = $router->chat($messages);
// simple prompts go to 'fast', complex ones to 'smart'
```

`ModelRouter` implements `ProviderInterface`, so it drops in wherever a provider is expected.

## Routing Strategies

A strategy decides which tier handles a request. The library ships several:

- `TaskComplexityStrategy` scores the request (length, keywords, tool count, multi-modal input) and routes simple work to a fast tier and complex work to a stronger one.
- `TokenLengthStrategy` routes based on the input token count against a threshold.
- `KeywordStrategy` routes when the prompt contains configured keywords.
- `AlwaysStrategy` sends everything to one fixed tier.

You can also add explicit rules or force a specific tier for a single call, which is handy for testing or overrides.

## Choosing Between Them

Reach for aliases when you simply want stable names for models and expect to swap the underlying IDs over time. Reach for the router when you want the library to select a provider or tier per request, for example to save cost on easy prompts while reserving a stronger model for hard ones. The two compose well: a router's tiers can themselves use aliases.

## Where to Next

See [Provider Fallback](learn/ai-provider-fallback) for resilience across providers, or [Basic Chat](learn/ai-basic-chat) for the options a routed call accepts.
