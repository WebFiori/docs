# Caching

The cache system provides a key-value store with TTL (time-to-live) expiration. It's used for reducing database load, storing computed results, and powering the rate limiter.

<meta name="description" content="Learn how to use WebFiori's cache system with FileStorage, TTL, and cache-aside pattern.">

## Basic Usage

Use `CacheFacade` for static access:

```php
use WebFiori\Cache\CacheFacade;

// Store a value (TTL in seconds)
CacheFacade::set('user:42', $userData, 300); // expires in 5 minutes

// Retrieve a value
$data = CacheFacade::get('user:42'); // returns null if expired/missing

// Check existence
CacheFacade::has('user:42');

// Delete
CacheFacade::delete('user:42');

// Clear all
CacheFacade::flush();
```

## Cache-Aside Pattern

The `get()` method accepts a generator callable that populates the cache on miss:

```php
$posts = CacheFacade::get('posts:page:1', function () {
    // Only called if cache is empty or expired
    return $this->postRepo->findPublished(1, 10);
}, 120); // TTL: 120 seconds
```

This is the recommended pattern — avoids cache stampede and keeps code concise.

## Cache Invalidation

Invalidate when data changes:

```php
// After creating/updating/deleting a post
$repo->save($post);
CacheFacade::flush(); // clear all

// Or delete specific keys
CacheFacade::delete('posts:page:1');
CacheFacade::delete('posts:page:2');
```

## Storage Backends

The default backend is `FileStorage`. You can swap it:

```php
use WebFiori\Cache\Cache;
use WebFiori\Cache\CacheFacade;
use WebFiori\Cache\FileStorage;
use WebFiori\Cache\RedisStorage;

// Use Redis
$cache = new Cache(new RedisStorage($redisConnection), true, 'myapp:');
CacheFacade::setInstance($cache);
```

Available backends:
- `FileStorage` — stores cache entries as files (default)
- `RedisStorage` — uses Redis for distributed caching

Implement `Storage` interface for custom backends.

## Prefixed Cache

Use prefixes to namespace cache keys:

```php
$cache = CacheFacade::withPrefix('api:');
$cache->set('users', $data, 60); // actual key: api:users
```

## Enabling/Disabling

```php
CacheFacade::setEnabled(false); // bypass cache (useful for debugging)
CacheFacade::isEnabled();       // check status
```

## Purging Expired Entries

```php
$purged = CacheFacade::purgeExpired(); // returns count of removed entries
```

## HTTP Caching

For HTTP-level caching (ETag, Cache-Control), see the `HttpCacheMiddleware` in the [Built-in Middleware](learn/built-in-middleware) guide.

## Route Caching

In production, route discovery (namespace scanning via reflection) runs on every request. Route caching eliminates this overhead:

```bash
# Build the cache (run after deploy)
php webfiori routes:cache

# Clear the cache (run before re-caching)
php webfiori routes:clear
```

Enable via environment variable:
```
ROUTE_CACHE_ENABLED=true
```

The route cache uses the same storage backend as the application cache (file or Redis). See [Environment Variables](learn/env-vars) for configuration.

> **Note:** Always clear and rebuild route cache after deploying new services or changing routes.

## Best Practices

- Use the cache-aside pattern (`get()` with generator) to avoid stampedes
- Set appropriate TTLs — short for frequently changing data, long for static content
- Invalidate on write, not on read
- Use prefixes to avoid key collisions between features
- Monitor cache hit rates to tune TTLs
