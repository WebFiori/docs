# Caching

Caching stores responses so that repeated identical requests are served without another API call, cutting cost and latency. WebFiori AI caches by exact request match: the same messages, model, and relevant options produce a cache hit.

<meta name="description" content="Cache AI responses in WebFiori AI with CacheConfig and a pluggable cache, including TTL and temperature-based skipping.">

## Enabling the Cache

Attach a cache implementation and a `CacheConfig` to the client:

```php
use WebFiori\Ai\Cache\CacheConfig;
use WebFiori\Ai\Cache\InMemoryCache;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Message;

$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...', model: 'gpt-4o-mini'));

$client->setCache(new InMemoryCache());
$client->setCacheConfig(new CacheConfig(
    enabled: true,
    defaultTtl: 3600,                 // keep entries for one hour
    skipCacheAboveTemperature: 0.5,   // do not cache creative responses
));

$messages = [new Message('user', 'What is the capital of France?')];

$client->chat($messages, ['temperature' => 0]); // API call, result cached
$client->chat($messages, ['temperature' => 0]); // cache hit, no API call
```

A different question, model, or a changed option produces a cache miss and a fresh call.

## Configuration Options

`CacheConfig` controls when and how long responses are cached:

- `enabled` turns caching on or off.
- `defaultTtl` is the time-to-live in seconds for chat responses.
- `embeddingTtl` sets a separate lifetime for cached embeddings.
- `skipCacheAboveTemperature` disables caching when the request temperature is above the given value. This matters because a high temperature is meant to produce varied output, so serving a cached reply would defeat the purpose. Set it to `null` to cache regardless of temperature.

## Choosing a Cache Backend

`InMemoryCache` lives for the current process, which is ideal for a single script, a worker, or tests. For caching that survives across web requests, implement `CacheInterface` over a shared store such as Redis, a database, or the filesystem, and pass your implementation to `setCache()`. Your calling code does not change.

```php
use WebFiori\Ai\Cache\CacheInterface;

final class RedisCache implements CacheInterface {
    // get(), set(), has(), delete(), clear()
}

$client->setCache(new RedisCache());
```

## What Gets Cached

Caching is best suited to deterministic requests (typically `temperature: 0`) where the same input should always yield the same output, such as classification, extraction, or fixed lookups. Free-form creative generation is a poor fit, which is what `skipCacheAboveTemperature` guards against. Requests that execute tools are not cached, since tools may have side effects.

## Where to Next

See [Observability](learn/ai-observability) to track cache hits and misses through metrics, or [Embeddings](learn/ai-embeddings) which can also be cached to avoid re-embedding the same text.
