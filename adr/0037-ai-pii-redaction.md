# ADR-0037: AI Library PII Redaction in Logs and Metrics

**Date:** 2026-08-11
**Status:** Accepted

## Context

AI requests often contain sensitive user data (names, emails, addresses, medical info). If this data reaches logging or metrics callbacks, it creates compliance violations (GDPR, HIPAA, etc.).

## Decision

### Scope

Redaction applies to **both** logging (via `LoggerTrait`) and metrics (via `MetricsTrait`) before invoking developer callbacks. Consistent behavior prevents sensitive data from leaking through either channel.

### What Gets Redacted

| Data | Behavior |
|------|----------|
| API keys / Bearer tokens | Always redacted — mandatory, non-configurable |
| Request/response body content | Opt-in — developer enables per `RedactionConfig` |
| Error messages | Yes — redacted using same rules (API errors may echo user input) |
| Metadata (model, latency, token counts, status codes) | Never redacted — not sensitive |

### Architecture

A dedicated `RedactionService` holds all redaction logic. Both `LoggerTrait` and `MetricsTrait` use it before invoking callbacks. Single responsibility, testable in isolation, no duplication.

```php
class RedactionService {
    public function redactString(string $text): string;
    public function redactContext(array $context): array;
}
```

### RedactionRule

Named rules for both built-in and custom patterns:

```php
new RedactionRule(
    name: 'email',
    pattern: '/\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b/i',
    replacement: '[EMAIL]'
)
```

### RedactionConfig

```php
$provider->setRedactionConfig(new RedactionConfig(
    redactRequestBodies: false,   // opt-in, default false
    redactResponseBodies: false,  // opt-in, default false
    disabledRules: ['email'],     // per-rule disable of built-ins
    customRules: [
        new RedactionRule('ssn', '/\b\d{3}-\d{2}-\d{4}\b/', '[SSN]'),
    ],
));
```

### Built-in Rules

| Rule Name | Pattern | Replacement | Disableable |
|-----------|---------|-------------|-------------|
| `api_key` | Common API key formats | `[API_KEY]` | ❌ No |
| `bearer_token` | Bearer token values | `[TOKEN]` | ❌ No |
| `email` | Email addresses | `[EMAIL]` | ✅ Yes |
| `phone` | Phone numbers | `[PHONE]` | ✅ Yes |
| `credit_card` | Credit card numbers | `[CC]` | ✅ Yes |

### Performance

When no `RedactionConfig` is set, all redaction is skipped (single null check). No regex overhead unless explicitly configured.

## Alternatives Considered

### Apply redaction in LoggerTrait and MetricsTrait directly
Duplicates logic across two traits, harder to test, diverges over time.

### Redact all log context fields
Over-aggressive — makes logs useless. Metadata like latency and model name contain no PII.

### Leave to developer's logging callback
Easy to forget. The library knows where sensitive data is — API keys are in headers, user content is in message bodies.

## Consequences

### Easier
- GDPR/HIPAA compliance for logged AI interactions
- Consistent redaction across logs and metrics
- Developers can add domain-specific rules (SSN, passport numbers, etc.)

### Harder
- API key and bearer token redaction is mandatory — cannot be disabled
- Request/response body logging requires opt-in (but this is intentional)
