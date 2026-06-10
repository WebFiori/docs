# ADR-0022: Email Instance is Single-Use (Not Reusable After Send)

**Date:** 2026-06-10
**Status:** Accepted

## Context

The `Email` class in `webfiori/mail` has an `isSent` flag that prevents `send()` from being called more than once on the same instance:

```php
if ($this->isSent()) {
    throw new SMTPException('Message was already sent.');
}
```

This raises a design question: should an `Email` object be reusable after sending? Three options were evaluated:

1. Add a `reset()` method that clears `isSent` and internal state, allowing the same instance to be sent again.
2. Call `reset()` automatically at the end of `send()`, making every Email implicitly reusable.
3. Keep the single-use design and decouple connection reuse from message reuse.

The question is motivated by performance: creating a new TCP+TLS+AUTH cycle per message is expensive. Users want to send many messages efficiently.

## Decision

**Keep `Email` as a single-use object. Do not add a `reset()` method.**

Connection reuse is solved separately by allowing `Email::send()` to accept an external `SMTPServer` instance (see mail#54) and by introducing `SMTPServer::reset()` (see mail#53) which sends the SMTP `RSET` command to reuse the connection for a new message.

The intended pattern is:

```php
$server = new SMTPServer('smtp.example.com', 465);
$server->authLogin('user', 'pass');

foreach ($recipients as $recipient) {
    $email = new Email($smtpAccount);
    $email->to($recipient)->subject('Hello')->send($server);
    $server->reset();
}

$server->disconnect();
```

Each message is a fresh `Email` instance with clean state. The connection is reused across messages via the shared `SMTPServer`.

## Alternatives Considered

### Option 1: Add `Email::reset()`

```php
$email->send();
$email->reset();
$email->removeAllRecipients();
$email->addTo('next@example.com');
$email->send();
```

**Rejected because:**
- Error-prone. The user must remember to clear recipients, but the HTML body, attachments, callbacks, and boundary hash all carry over. Forgetting to clear any one of these causes subtle bugs (wrong content sent to wrong recipient, duplicate attachments).
- The `beforeSendPool` and `afterSendPool` have `executed` flags that must be re-armed — easy to get wrong.
- The boundary hash is derived from `date()` at construction time. Reusing it across messages may cause deduplication issues on receiving servers.

### Option 2: Auto-reset inside `send()`

```php
public function send() {
    // ... send logic ...
    $this->resetState(); // auto-reset at the end
}
```

**Rejected because:**
- Removes the double-send safety guard. Accidental `$email->send(); $email->send();` in a loop or callback silently sends the message twice.
- The `isSent()` method becomes meaningless — it always returns `false` after send completes.
- Violates principle of least surprise: calling `send()` should be a terminal operation, not a cycle.

### Option 3 (chosen): Single-use Email, reusable connection

**Advantages:**
- Each `Email` has clean, isolated state — no risk of leaking data between messages.
- The `isSent` guard prevents accidental double-delivery.
- Connection reuse is handled at the transport layer (`SMTPServer`), which is the correct abstraction boundary.
- Constructing a new `Email` is cheap (no I/O, just object allocation).
- Natural PHP pattern — objects are lightweight and GC handles cleanup.

## Consequences

### Benefits

- Clear ownership: one `Email` = one message = one delivery attempt.
- No hidden state leaks between messages.
- Double-send protection remains intact.
- Connection pooling (mail#55) works naturally — pool manages `SMTPServer` instances, each `Email` is stateless with respect to transport.

### Trade-offs

- Slightly more verbose for bulk sending (must create new `Email` per message). Mitigated by the fluent API making construction concise.
- Users who want to reuse a "template" Email must construct a factory function or builder pattern themselves.

### Related

- mail#53: `SMTPServer::reset()` — enables SMTP-level connection reuse via `RSET`
- mail#54: `Email::send()` accepts external `SMTPServer` — decouples message from transport
- mail#55: `SMTPConnectionPool` — high-level pool built on top of #53 and #54
