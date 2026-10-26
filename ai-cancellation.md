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

## Observability

A cancelled request is recorded as a **cancellation**, not a failure: a
`request.cancelled` metric, a `Status::CANCELLED` status event, and an audit
entry with status `cancelled`. Streaming cancellation emits a `stream.cancelled`
metric.

## Guarantee

Cancellation is **best-effort, near-immediate** (~sub-second): the request is
aborted and the socket torn down within a few hundred milliseconds of
`cancel()`, bounded by cURL's progress-callback cadence. It is a real teardown
that stops provider work and cost — but it is **not** a literally-instantaneous
kill. A hard/immediate guarantee would require a different transport model and
is out of scope.

## Notes

The cancellation primitive is deliberately **self-contained** and is not built
on `WebFiori/pubsub` (that package is still in progress); migrating to it later
is a documented future option. See
[ADR-0057](adr/0057-ai-request-cancellation.md) for the full decision record.

A runnable, offline example of all three outcomes is in
[`examples/33-cancellation`](https://github.com/WebFiori/ai/tree/main/examples/33-cancellation).
