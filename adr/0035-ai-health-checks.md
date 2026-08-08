# ADR-0035: AI Library Health Checks

**Date:** 2026-08-09
**Status:** Accepted

## Context

Production systems need to verify that AI providers are reachable and responding correctly. Load balancers, monitoring tools, and orchestration systems require a simple health check mechanism to:

- Detect provider outages quickly
- Route traffic away from unhealthy providers
- Alert operations teams when providers become unavailable

## Decision

### Health Check Method

Add `healthCheck()` to `ProviderInterface`:

```php
interface ProviderInterface {
    // ... existing methods ...
    
    public function healthCheck(int $timeout = 5): HealthCheckResult;
}
```

### HealthCheckResult

```php
class HealthCheckResult {
    public function isAvailable(): bool;
    public function getLatencyMs(): int;
    public function getError(): ?string;
    public function getCheckedAt(): \DateTimeInterface;
    public function getCheckMethod(): string;  // e.g., 'models_list', 'minimal_completion'
}
```

### Provider-Specific Implementation

| Provider | Check Method | Endpoint |
|----------|--------------|----------|
| OpenAI | `models_list` | `GET /v1/models` (no token cost) |
| Google | `models_list` | Model info endpoint |
| Anthropic | `minimal_completion` | Tiny completion (max_tokens: 1) |
| Bedrock | `models_list` | `ListFoundationModels` API |

For providers without free metadata endpoints, make a minimal completion request:
```php
messages: [{"role": "user", "content": "Hi"}]
max_tokens: 1
```
This costs ~$0.00001 per check but verifies full functionality.

### Timeout

Health checks use a separate, shorter timeout (default 5 seconds) rather than the provider's configured timeout:

```php
$status = $provider->healthCheck();           // 5s default
$status = $provider->healthCheck(timeout: 10); // custom timeout
```

Health checks should fail fast to quickly detect outages.

### Bypass Caching and Retry

Health checks bypass:
- **Caching** — A cached response doesn't reflect current availability
- **Retry logic** — Retries mask failures and add latency

The check should give raw, real-time status.

### Error Handling

`healthCheck()` never throws. Errors are captured in the result:

```php
$status = $provider->healthCheck();
if (!$status->isAvailable()) {
    echo "Provider down: " . $status->getError();
}
```

## Alternatives Considered

### TCP ping only
Doesn't verify API functionality, authentication, or quota.

### Full completion request
Wastes tokens and money. Health checks may run frequently.

### Make health check optional (not in interface)
Production systems need this. All real providers can implement some form of check.

## Consequences

### Easier
- Load balancers can route based on provider health
- Monitoring systems can alert on outages
- Applications can implement fallback logic based on health

### Harder
- Custom `ProviderInterface` implementations must add `healthCheck()`
- Providers without metadata endpoints incur tiny costs per check
