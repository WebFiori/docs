# ADR-0036: AI Library Metrics Collection via Callback

**Date:** 2026-08-11
**Status:** Accepted

## Context

Enterprise applications need to track AI usage metrics: latency, token consumption, error rates, and costs. The library should emit structured metrics without requiring a specific monitoring tool.

## Decision

### Metrics Callback

Add a separate `setMetricsCallback()` to providers, following the same pattern as `setLogCallback()`:

```php
$provider->setMetricsCallback(function (string $event, array $data) {
    // Send to Prometheus, DataDog, CloudWatch, etc.
});
```

Kept separate from the logging callback — metrics are structured data for monitoring systems, logs are human-readable diagnostic messages.

### Request ID

Auto-generated per request using `uniqid('req_', true)`. Developers can override via options:

```php
$provider->chat($messages, ['request_id' => $myId]); // custom
$provider->chat($messages);                           // auto-generated
```

The request ID is included in every metric event and returned in responses for traceability:

```php
$response = $provider->chat($messages);
$response->getRequestId(); // correlate with metrics
```

### Emitted Events

All events include `timestamp`, `request_id`, `provider`, and `model`.

| Event | Data | Emitted by |
|-------|------|------------|
| `request.sent` | endpoint, method | All operations |
| `request.completed` | status_code, latency_ms, prompt_tokens, completion_tokens, total_tokens | chat, embed, image |
| `request.failed` | error_type, error_message, latency_ms | All operations |
| `request.retried` | attempt, delay_ms, reason | retry middleware |
| `rate_limit.approaching` | remaining, resets_at | rate limit middleware |
| `rate_limit.hit` | retry_after_ms | rate limit middleware |
| `cache.hit` | key | chat, embed |
| `cache.miss` | key | chat, embed |
| `stream.started` | — | streamChat |
| `stream.completed` | duration_ms, tokens | streamChat |
| `stream.error` | error | streamChat |
| `health_check.completed` | latency_ms, check_method | healthCheck |
| `health_check.failed` | error, latency_ms, check_method | healthCheck |

### Coverage

Metrics are emitted for all operations: `chat()`, `streamChat()`, `embed()`, `generateImage()`, and `healthCheck()`.

### Emission Model

Synchronous only. Async emission is the developer's responsibility inside their callback.

## Alternatives Considered

### OpenTelemetry dependency
Too heavy, not PHP-native enough. Conflicts with zero-dependency principle.

### Unified logging + metrics callback
Metrics are structured data for machines; logs are for humans. Keeping them separate avoids conflating concerns and allows independent configuration.

### Async emission
Would require ReactPHP, fibers, or an async abstraction. Conflicts with zero-dependency principle. Developers can implement async inside their own callback.

## Consequences

### Easier
- Developers can bridge to any monitoring system (Prometheus, DataDog, CloudWatch, etc.)
- Full cost visibility across all operations including embeddings and image generation
- Request ID enables end-to-end tracing across metrics, logs, and application code

### Harder
- All response DTOs need a `request_id` field (minor addition)
- `AbstractClient` needs to propagate request IDs through the call chain
