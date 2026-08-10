# ADR-0038: AI Library Structured Audit Logging

**Date:** 2026-08-11
**Status:** Accepted

## Context

Enterprises need an audit trail of all AI interactions for compliance, cost allocation, and incident investigation. General logging (#5) is insufficient because audit has stricter requirements: always on, fully structured, and never filtered by log level.

## Decision

### Audit Callback

A separate `setAuditCallback()` distinct from the general log callback:

```php
$provider->setAuditCallback(function (array $entry) {
    // Write to audit table, SIEM, etc.
});
```

### Audit Entry Structure

```php
[
    'request_id'  => 'req_abc123',
    'timestamp'   => 1786399606698,      // Unix ms
    'operation'   => 'chat',             // chat | embed | generateImage | streamChat
    'provider'    => 'openai',
    'model'       => 'gpt-4o',
    'status'      => 'success',          // success | error | rate_limited
    'duration_ms' => 1234,
    'tokens'      => [
        'prompt'     => 150,
        'completion' => 89,
        'total'      => 239,
    ],
    'error'    => null,                  // error details on failure
    'messages' => [...],                 // opt-in, PII-redacted
    'response' => '...',                 // opt-in, PII-redacted
    'metadata' => ['user_id' => 'u-123', 'tenant_id' => 't-456'],
]
```

### Audit Context

Both static and dynamic, merged at emit time:

```php
// Static — set once (e.g., at middleware level)
$provider->setAuditContext(['tenant_id' => 'tenant-456']);

// Dynamic — per-request override/merge
$provider->chat($messages, [
    'audit_context' => ['user_id' => 'user-123'],
]);

// Audit entry gets both: tenant_id + user_id
```

### AuditConfig

Controls opt-in content inclusion:

```php
$provider->setAuditConfig(new AuditConfig(
    includeMessages: true,   // include full request messages (default: false)
    includeResponse: true,   // include AI response content (default: false)
));
```

### PII Redaction

Audit entries are always processed through PII redaction (if configured) before reaching the callback. This is mandatory — not optional.

### Coverage

| Operation | Audited |
|-----------|---------|
| `chat()` | ✅ |
| `streamChat()` | ✅ Single entry after completion |
| `embed()` | ✅ |
| `generateImage()` | ✅ |
| `healthCheck()` | ❌ Infrastructure probe, not a business event |

### Timestamp

Unix timestamp in milliseconds — consistent with metrics events.

## Alternatives Considered

### Combine with logging callback
Audit has different requirements: always on, structured, never filtered by level. Keeping it separate avoids conflation.

### Include healthCheck() in audit
Health checks are infrastructure probes, not business transactions. They run frequently (every 30s in load balancers) and would flood the audit trail with low-value noise. Metrics already cover health check observability.

### Start + end entries for streamChat()
A single post-completion entry has complete data (duration, tokens, response). Two entries would require correlation logic in audit consumers.

### ISO 8601 timestamp
Unix milliseconds are consistent with metrics events and easier to sort/compare in databases.

## Consequences

### Easier
- Full compliance audit trail for GDPR, HIPAA, PDPL
- Cost allocation per user/tenant via metadata
- Incident investigation with full request/response (opt-in)
- PII-safe by default — body content requires explicit opt-in

### Harder
- Body content requires enabling both `AuditConfig` and `RedactionConfig` for safe storage
- Developers must set audit context at the appropriate scope (middleware vs per-request)
