# ADR-0023: Transport Abstraction with Backward-Compatible Introduction

**Date:** 2026-06-10
**Status:** Accepted

## Context

The `Email` class is responsible for both composing the message (HTML body, recipients, headers) and delivering it (SMTP connection, authentication, protocol commands). This tight coupling means:

- Cannot swap delivery backends (e.g. SES API, SendGrid) without rewriting `Email`.
- Cannot test message composition without an SMTP server.
- SMTP protocol fixes (pooling, retry, timeout) must be done inside `Email`, bloating it further.

A transport abstraction is needed to separate concerns. Several design decisions were made:

1. What form should the abstraction take?
2. How to introduce it without breaking existing users?
3. Should DSN (URI-style connection strings) be supported?

## Decision

### 1. Interface over abstract class

Define a `TransportInterface` with a single `send(Email $message): void` method.

An interface was chosen over an abstract class because:
- It imposes no inheritance constraints on implementors.
- Transports vary wildly (socket-based SMTP vs HTTP API vs file write) — shared base logic would be forced.
- PHP supports multiple interface implementation but single inheritance.

### 2. Non-breaking introduction with deprecation path

`Email::send()` gains an optional `?TransportInterface $transport` parameter:

```php
public function send(?TransportInterface $transport = null): void {
    if ($transport !== null) {
        $transport->send($this);
    } else {
        // Existing SMTP logic (deprecated)
        $this->sendViaSmtp();
    }
}
```

- **v2.x:** Both paths work. Calling `send()` without a transport triggers a deprecation notice.
- **v3.0:** Transport parameter becomes required. Internal SMTP logic is removed from `Email`.

This was chosen over making transport required immediately because every existing user calls `$email->send()` without arguments.

### 3. No DSN parser

A DSN parser (`smtps://user:pass@host:465`) was considered and rejected because:

- The library currently has only one transport (SMTP).
- The existing `SMTPAccount` + `AccountOption` API is explicit, type-safe, and IDE-friendly.
- DSN parsing adds complexity (format documentation, error handling for malformed strings) with no benefit when there's one backend.
- If multiple transports are added in the future, DSN can be layered on top as a factory without changing the transport interface.

## Alternatives Considered

### Abstract class instead of interface

Rejected — no meaningful shared logic between an SMTP socket transport and an HTTP API transport.

### Callable/closure instead of interface

```php
$email->send(function (Email $msg) { /* custom logic */ });
```

Rejected — loses type safety, IDE autocompletion, and the ability to configure the transport as an object with state (connection pooling, retry config).

### Breaking introduction (transport required immediately)

Rejected — forces all users to change `$email->send()` to `$email->send(new SmtpTransport($account))` in a minor version. Inappropriate for a library with existing users.

### Include DSN support from the start

Rejected — YAGNI. One transport doesn't need a routing mechanism. Can be added later without breaking changes.

## Consequences

### Benefits

- `Email` becomes purely a message builder — simpler, easier to test.
- New transports (SES, SendGrid, Null, File) can be added without touching core classes.
- Connection pooling (#55) and retry logic (#56) live in `SmtpTransport`, not `Email`.
- Testing becomes trivial with a `NullTransport`.

### Trade-offs

- Deprecation period means maintaining two code paths in v2.x.
- Users must eventually migrate to explicit transport (in v3.0).
- No DSN means no single-string configuration — users must construct objects. This is acceptable for the library's scope.

### Related

- mail#57: TransportInterface definition
- mail#58: SmtpTransport extraction
- mail#53: SMTPServer::reset() (used internally by SmtpTransport)
- mail#55: Connection pool (built on SmtpTransport)
- ADR-0022: Email single-use instance (transport abstraction doesn't change this)
