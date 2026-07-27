# ADR-0032: SMTP Stream Timeout and Connection Retry with Exponential Backoff

**Date:** 2026-07-27
**Status:** Accepted

## Context

`SMTPServer` has a `$responseTimeout` property (default: 5 minutes) that is only applied to the initial TCP connection establishment via `stream_socket_client()`'s timeout parameter. After the connection is open, no timeout is applied to subsequent reads. The `read()` method calls `fgets()` in a loop with no deadline, meaning if the server becomes unresponsive mid-session (network partition, server crash, firewall silently dropping packets), the PHP process blocks indefinitely.

Additionally, transient connection failures — a server temporarily refusing connections, a brief network hiccup, a slow server not yet ready — cause `connect()` to fail immediately with no retry. In long-running workers (queue consumers, scheduled tasks) this means a momentary blip permanently kills the worker's ability to send email until it is manually restarted.

Two decisions needed to be made:

1. How to apply a timeout to post-connection reads.
2. Whether and how to retry failed connections.

## Decision

### 1. Apply stream timeout immediately after connection

After a successful `stream_socket_client()` or `fsockopen()` call, call `stream_set_timeout()` on the stream resource using the same `$responseTimeout` value (converted to seconds):

```php
stream_set_timeout($conn, $this->responseTimeout * 60);
```

In `read()`, after each `fgets()` call, check `stream_get_meta_data()` for the `timed_out` flag and throw `SMTPException` immediately if it is set:

```php
$meta = stream_get_meta_data($this->serverCon);
if ($meta['timed_out']) {
    throw new SMTPException(
        'SMTP read timed out after ' . ($this->responseTimeout * 60) . 's.',
        0,
        $this->getLog()
    );
}
```

**A timed-out read is not retried in place.** When `fgets()` times out, the SMTP session state is unknown — the server may have partially sent a response, may be mid-command, or may be gone entirely. Resuming the read or resending the last command would produce undefined behavior. The exception propagates to the caller, who must decide whether to reconnect from scratch.

### 2. Retry initial connection with exponential backoff

Wrap the `connect()` logic in a retry loop. On each failed attempt, sleep for `baseDelay * 2^(attempt-1)` seconds before retrying. With the defaults (3 retries, 1s base delay):

| Attempt | Wait before |
|---------|-------------|
| 1       | none        |
| 2       | 1s          |
| 3       | 2s          |
| 4       | 4s          |

`maxRetries` and `retryBaseDelay` are exposed as constructor parameters on `SMTPServer`, and as `AccountOption::MAX_RETRIES` / `AccountOption::RETRY_DELAY` on `SMTPAccount`, so callers can tune them.

**EHLO rejection after a successful TCP connection is not retried.** If the server accepted the TCP connection but rejected `EHLO`, the server is functioning and has made a deliberate decision. Retrying immediately will produce the same result. The connection is broken and the failure is surfaced immediately.

**Read timeouts are not retried.** Covered above — session state is unknown after a timeout.

### 3. Retry scope is connection establishment only

The retry boundary is: everything up to and including the initial `EHLO` handshake and STARTTLS upgrade. Authentication (`AUTH LOGIN`, `AUTH XOAUTH2`) and message sending are not retried. This keeps retry logic simple and scoped to the part of the lifecycle where transient failures are realistic and safe to retry.

## Alternatives Considered

### Retry on read timeout

Rejected. A read timeout means the server stopped responding mid-session. The SMTP session state is undefined at that point — the server may have buffered a partial command or may be in an indeterminate state. Retrying a `RCPT TO` or `DATA` command after a timeout could result in duplicate sends or protocol errors. The only safe recovery is to tear down the connection and start over from the top, which is the caller's responsibility.

### Non-blocking I/O with `stream_select()`

`stream_select()` could poll the stream with a timeout without blocking. This would avoid the blocking `fgets()` problem without relying on `stream_set_timeout()`.

Rejected for this change — `stream_select()` would require restructuring `read()` significantly and complicates the code considerably for the same observable outcome. `stream_set_timeout()` + `stream_get_meta_data()` is the idiomatic PHP approach for this pattern and is well-understood.

### Retry with jitter

Adding random jitter to the backoff delay (e.g. `delay + rand(0, delay)`) reduces thundering herd problems when many workers reconnect simultaneously after a server restart.

Not included in the initial implementation — the library is typically used with one connection per email send, not a large pool of workers. Can be added later if connection pooling (#55) is implemented and makes thundering herd a realistic concern.

### Configurable retry predicate (callback)

Allow callers to supply a callable that decides whether to retry based on the error:

```php
$server->setRetryPredicate(fn($attempt, $error) => $attempt < 3);
```

Rejected — over-engineered for the scope. The retry condition is simple and fixed: retry if the TCP connection failed, don't retry otherwise. A predicate adds API surface and cognitive overhead without solving a real user problem.

### `sleep()` vs async

Using `sleep()` for backoff is a blocking call that ties up the PHP process. In async contexts (ReactPHP, Swoole, Fibers) this is problematic.

Accepted as a known trade-off. The library uses synchronous blocking I/O throughout (`stream_socket_client`, `fgets`, `fwrite`). Introducing async-aware delays while keeping everything else synchronous would be inconsistent. If async support is added in the future, the retry logic would need to be revisited as part of a broader async overhaul.

## Consequences

### Benefits

- PHP processes (especially CLI workers) no longer hang indefinitely when an SMTP server becomes unresponsive.
- Transient connection failures in queue workers recover automatically without manual intervention.
- `$responseTimeout` now applies consistently to the entire session, not just connection establishment.
- Retry behavior is configurable per-account for callers with different reliability requirements.

### Trade-offs

- `sleep()` in the retry loop blocks the process during backoff. Total worst-case blocking time with defaults: 1 + 2 + 4 = 7 seconds before `connect()` gives up.
- Read timeout exceptions add a new failure mode callers must handle. Existing code that does not catch `SMTPException` on `send()` will surface this as an uncaught exception rather than a hang — a better failure mode, but a behavioral change.

### Related

- mail#50: The bug this addresses
- mail#55: Connection pooling — retry and timeout behavior will need to be coordinated with pool-level reconnect logic
