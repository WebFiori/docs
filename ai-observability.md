# Observability

WebFiori AI gives you three ways to see what your AI calls are doing in production: metrics for monitoring, audit logs for compliance, and health checks for availability. Each is opt-in through a callback or config, so nothing is emitted until you ask for it.

<meta name="description" content="Monitor WebFiori AI with metrics callbacks, structured audit logging, and provider health checks.">

## Metrics

Register a metrics callback to receive structured events such as requests sent, requests completed, cache hits, and errors. Forward them to whatever you use (StatsD, Prometheus, DataDog, logs):

```php
$client->setMetricsCallback(function (string $event, array $data): void {
    // $event: e.g. 'request.sent', 'request.completed', 'cache.hit'
    // $data:  provider, model, latency_ms, token counts, etc.
    statsd_increment($event);
    if (isset($data['latency_ms'])) {
        statsd_timing($event.'.latency', $data['latency_ms']);
    }
});

$client->chat([new Message('user', 'Hello')]);
```

The callback fires around each operation, so you can track throughput, latency, token usage, and cache effectiveness without touching your business logic.

## Audit Logging

Audit logging records a structured entry for every operation, which is useful for compliance and after-the-fact investigation. Provide a callback, and optionally an `AuditConfig` to control how much is captured:

```php
use WebFiori\Ai\Audit\AuditConfig;

$client->setAuditCallback(function (array $entry): void {
    // $entry: operation, provider, model, status, duration_ms, tokens, error, ...
    file_put_contents('audit.log', json_encode($entry).PHP_EOL, FILE_APPEND);
});

// By default, message content and responses are NOT included.
// Opt in explicitly when your compliance needs require it:
$client->setAuditConfig(new AuditConfig(
    includeMessages: true,
    includeResponse: true,
));
```

Message content and responses are excluded by default so you do not log sensitive text unintentionally. Combine audit logging with [PII redaction](learn/ai-security) when you do include content.

## Health Checks

`healthCheck()` verifies a provider is reachable and responding. It never throws; the outcome is captured in a `HealthCheckResult`, and the check bypasses caching and retry logic to give a real-time answer:

```php
$result = $client->healthCheck(timeout: 10);

if ($result->isAvailable()) {
    echo 'OK in '.$result->getLatencyMs().' ms';
} else {
    echo 'Unavailable: '.$result->getError();
}
```

This suits readiness probes and dashboards. For a `FallbackProvider` or `ModelRouter`, the health check aggregates across the underlying providers.

## Redaction

Metrics and audit context are run through the redaction service, so credentials and common PII patterns are scrubbed before they reach your callbacks. See [Security and PII Redaction](learn/ai-security) for the details and how to add custom rules.

## Where to Next

See [Security and PII Redaction](learn/ai-security) to protect sensitive data in logs, or [Provider Fallback](learn/ai-provider-fallback) to act on health and failure signals.
