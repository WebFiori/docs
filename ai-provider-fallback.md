# Provider Fallback

`FallbackProvider` wraps several provider clients and automatically fails over to the next one when a call fails. It implements `ProviderInterface`, so your application code treats it exactly like a single provider while gaining resilience across OpenAI, Google, Anthropic, and Bedrock.

<meta name="description" content="Add resilience to WebFiori AI with FallbackProvider: automatic failover across providers, fallback strategies, and a circuit breaker.">

## Basic Failover

Pass an ordered list of providers. `FallbackProvider` tries them in turn until one succeeds:

```php
use WebFiori\Ai\Provider\Fallback\FallbackProvider;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Provider\Anthropic\AnthropicClient;
use WebFiori\Ai\Provider\Anthropic\AnthropicClientConfig;
use WebFiori\Ai\Message;

$openai    = new OpenAIClient(new OpenAIClientConfig(apiKey: '...', model: 'gpt-4o'));
$anthropic = new AnthropicClient(new AnthropicClientConfig(apiKey: '...', model: 'claude-sonnet-4-20250514'));

$provider = new FallbackProvider([$openai, $anthropic]);

$response = $provider->chat([new Message('user', 'Hello!')]);
echo $response->getMessage()->getContent();
echo $provider->getLastUsedProvider();   // which provider answered
```

Because it satisfies `ProviderInterface`, you can pass a `FallbackProvider` anywhere a provider is expected, and `chat()`, `streamChat()`, `embed()`, and `generateImage()` all fail over.

## Configuring the Behavior

A `FallbackConfig` controls the strategy and how many providers to try:

```php
use WebFiori\Ai\Provider\Fallback\FallbackProvider;
use WebFiori\Ai\Provider\Fallback\FallbackConfig;
use WebFiori\Ai\Provider\Fallback\FallbackStrategy;

$provider = new FallbackProvider(
    providers: [$openai, $anthropic, $google],
    config: new FallbackConfig(
        strategy: FallbackStrategy::SEQUENTIAL,
        maxAttempts: 3,
    )
);
```

`FallbackStrategy` is a backed enum. `SEQUENTIAL` always tries providers in the given order, which is the common choice for a clear primary and backups. Other strategies distribute load across providers (for example round-robin), falling back on failure just the same.

## Circuit Breaker

To stop hammering a provider that keeps failing, attach a `CircuitBreakerConfig`. After a threshold of failures the breaker opens and that provider is skipped until a cooldown passes, then it is retried:

```php
use WebFiori\Ai\Provider\Fallback\FallbackConfig;
use WebFiori\Ai\Provider\Fallback\CircuitBreakerConfig;

$config = new FallbackConfig(
    strategy: FallbackStrategy::SEQUENTIAL,
    circuitBreaker: new CircuitBreakerConfig(
        failureThreshold: 5,
        cooldownSeconds: 30,
        successThreshold: 2,
    ),
);
```

The breaker moves through closed (normal), open (skipping the provider), and half-open (trial requests during recovery) states as failures and successes accumulate.

## When Everything Fails

If every provider fails, the last error propagates as a `ProviderException`, so wrap the call when you need a graceful degradation path:

```php
use WebFiori\Ai\Exception\ProviderException;

try {
    $response = $provider->chat([new Message('user', 'Hello!')]);
} catch (ProviderException $e) {
    // all providers exhausted; show a friendly message or queue for retry
}
```

## Where to Next

See [Model Routing](learn/ai-model-routing) to pick providers by request characteristics, or [Observability](learn/ai-observability) to monitor failures and health.
