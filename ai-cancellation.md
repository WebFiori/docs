# Request Cancellation

Cancellation lets you abort an in-flight AI request. Unlike throwing from a
streaming callback, it performs a **real teardown**: the HTTP request to the
provider is aborted and the socket closed, so the model stops working and token
cost stops accruing. It is aware of the auto-execute tool loop, and for
streaming it surfaces the partial response generated so far.

<meta name="description" content="Cancel in-flight WebFiori AI requests with CancellationToken: real socket teardown, tool-loop aware, partial response on streaming cancel, via ChatOption::CANCELLATION_TOKEN.">

> **Since 1.5.0**

## The primitive: `CancellationTokenSource`

You create a `CancellationTokenSource`, pass its token into a request, and call
`cancel()` from wherever the "stop" signal originates (a UI button, a detected
client disconnect, a timeout). The token is a self-contained pub/sub primitive —
multiple `onCancel` listeners may subscribe, and all fire on cancellation so
each layer can react (HTTP teardown, tool loop, application cleanup, logging).

```php
use WebFiori\Ai\Cancellation\CancellationTokenSource;

$source = new CancellationTokenSource();
$token  = $source->getToken();

$token->onCancel(function (): void {
    error_log('AI request cancelled');
});

// ... later, from another signal:
$source->cancel();
```

Semantics:

- Listeners registered **after** `cancel()` already fired run **immediately**.
- A listener that throws does **not** prevent the other listeners from running.
- `cancel()` is **idempotent** — listeners fire exactly once.

## Passing the token to a request

The token travels through the existing `$options` array under
`ChatOption::CANCELLATION_TOKEN` — there are no method-signature changes.

```php
use WebFiori\Ai\ChatOption;

$response = $client->chat($messages, [
    ChatOption::CANCELLATION_TOKEN => $token,
]);
```

## Non-streaming cancellation

A cancelled non-streaming `chat()` throws `OperationCancelledException` (distinct
from a transport `HttpException`):

```php
use WebFiori\Ai\Exception\OperationCancelledException;

try {
    $response = $client->chat($messages, [
        ChatOption::CANCELLATION_TOKEN => $token,
    ]);
} catch (OperationCancelledException $e) {
    // The request — and its token cost — was stopped.
}
```

## Streaming cancellation and the partial response

For `streamChat()`, cancellation stops token delivery and invokes an `onCancel`
callback — passed via `ChatOption::ON_CANCEL` — with the **accumulated partial
`ChatResponse`** (the text seen so far, token count, model). On cancel,
`onComplete` is **not** called, and cancellation is **not** surfaced through
`onError`.

```php
use WebFiori\Ai\ChatOption;
use WebFiori\Ai\ChatResponse;

$client->streamChat(
    $messages,
    function (string $chunk) use ($source): void {
        echo $chunk;
        flush();

        if (connection_aborted() === 1) {
            $source->cancel();   // client disconnected — stop the provider
        }
    },
    onComplete: null,
    onError: null,
    options: [
        ChatOption::CANCELLATION_TOKEN => $token,
        ChatOption::ON_CANCEL => function (ChatResponse $partial): void {
            // Persist or assess whatever was generated before the stop.
            savePartial($partial->getMessage()->getContent());
        },
    ],
);
```

If cancellation lands before the first token (during initial latency), the
partial response carries empty text but valid structure.

## Tool-loop cancellation

When `auto_execute_tools` is enabled, `chat()` may make several provider calls
per turn, interleaved with tool execution. Cancellation is checked at the top of
each iteration, before each tool runs, and after tool execution — so a cancel
during a long multi-tool turn stops promptly rather than only at the next HTTP
boundary. The tool loop surfaces `OperationCancelledException`.

If the outer token is cancelled while a tool is running, an exception thrown by
that tool (including `OperationCancelledException`) stops the run; it is not
recorded as a tool failure. A tool that throws `OperationCancelledException`
for its own reasons (for example its own deadline token) while the outer token
is still active is treated as an ordinary tool failure and the model continues.

`AgentTool` receives the outer run's token for the duration of its execution
and passes it to the sub-agent's `chat()` call, so cancelling the outer run also
aborts the nested one. The outer token takes precedence over a token supplied in
the tool's constructor options.

Retry backoff (`RetryableHttpClient`) also observes the token: a cancel during
the wait before a retry stops the retry promptly instead of after the full
delay, and a cancellation is never retried.

## Stopping from another request (external stop signal)

`cancel()` runs in the process that calls it. A running `chat()` is blocked
inside `curl_exec`, so a "stop" issued from a **different** HTTP request cannot
call `cancel()` on that request's source. To support this, give the source a
poller that reports whether a stop was requested, for example by reading a flag
the stop request wrote to a shared store:

```php
$source = new CancellationTokenSource(
    poller: fn (): bool => $store->isStopRequested($turnId),
    pollIntervalMs: 500,
    logCallback: $logger, // function (string $level, string $message, array $context)
);

$client->chat($messages, [
    ChatOption::CANCELLATION_TOKEN => $source->getToken(),
]);
```

`isCancelled()` (and so `throwIfCancelled()` and every library checkpoint)
consults the poller at most once per `pollIntervalMs`. When it returns `true`
the source is cancelled through the normal path: listeners fire once.

- The poller must be **cheap**; it can run inside the cURL progress callback.
- An exception thrown by the poller is treated as "not cancelled". It is
  reported through `logCallback` once per failure streak (not on every poll), so
  a broken store does not stay silent. Pass a log callback if the stop button
  matters to you.
- The poller only runs when a checkpoint calls `isCancelled()`. A request that
  carries a token runs on a `curl_multi` loop that returns control to PHP about
  every 100 ms, so the token (and the poller, subject to its interval) is
  checked at that cadence even while the provider sends nothing. A longer
  interval adds up to that interval of delay.
- The interval defaults to 500 ms. An injectable `clock` (milliseconds) is
  available for deterministic tests.
- Measured against a local server that stays idle for 12 s (libcurl 8.5.0): with
  a 50 ms interval a stop written by another process was noticed after 21–86 ms,
  streaming and non-streaming alike. Against the Anthropic API, a stop requested
  1.0 s into a long non-streaming request aborted it after 1.02 s. The worst
  case is roughly the poll interval plus ~100 ms; the default of 500 ms is a
  reasonable balance between promptness and the cost of the poller.
- If the poller reads a flag **file**, call `clearstatcache(true, $path)` first.
  PHP caches `filesize()`/`is_file()` results within a request, so a flag
  written while the request is running can otherwise go unnoticed. Cache and
  database stores are not affected.
- Without a poller the behaviour is unchanged.

## Observability

A cancelled request is recorded as a **cancellation**, not a failure: a
`request.cancelled` metric, a `Status::CANCELLED` status event, and an audit
entry with status `cancelled`. Streaming cancellation emits a `stream.cancelled`
metric.

## Guarantee

Cancellation is **best-effort, near-immediate**: a request that carries a token
is run on a `curl_multi` loop that re-checks the token about every 100 ms, so the
request is aborted and the socket torn down roughly 100 ms after `cancel()` (or,
with a poller, after the poller reports a stop: roughly the poll interval plus
100 ms). Measured on libcurl 8.5.0 against an idle server: 21–86 ms with a 50 ms
poll interval. `cancel()` still only takes effect when it is called in the
process that is waiting on the request; a stop issued from a different HTTP
request needs a poller. It is a real teardown that stops provider work and cost,
but it is **not** a literally-instantaneous kill. Requests without a token run
on a plain blocking `curl_exec` as before.

## Notes

The cancellation primitive is deliberately **self-contained** and is not built
on `WebFiori/pubsub` (that package is still in progress); migrating to it later
is a documented future option. See
[ADR-0057](adr/0057-ai-request-cancellation.md) for the full decision record.

A runnable, offline example of all three outcomes is in
[`examples/33-cancellation`](https://github.com/WebFiori/ai/tree/main/examples/33-cancellation).
