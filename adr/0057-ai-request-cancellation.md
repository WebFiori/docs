# ADR-0057: AI: First-class request cancellation (`CancellationToken`)

**Date:** 2026-10-25
**Status:** Proposed

## Context

An in-flight AI request cannot be cancelled. `ProviderInterface::chat()` and
`streamChat()` run to completion, and the only "stop" available is cooperative —
throwing from a streaming `onToken` callback. That is insufficient:

- it does nothing during the initial latency before the first token, nor during
  tool execution;
- it does **not** tear down the underlying HTTP request to the model, so the
  provider keeps working and the token cost keeps accruing after the user has
  abandoned the request;
- the auto-execute tool loop (`AbstractClient`) makes several HTTP calls per
  turn with no cancellation checkpoint between them.

Callers (e.g. a chat UI with a "stop" button, or a server detecting
`connection_aborted()`) need a **true** cancellation: abort the request, close
the socket to the provider, stop the tool loop, and let interested parties run
cleanup.

The transport is cURL (`CurlHttpClient`): non-streaming uses a blocking
`curl_exec` with `CURLOPT_RETURNTRANSFER`; streaming uses
`CURLOPT_WRITEFUNCTION` with `RETURNTRANSFER=false` (and already aborts by
returning `0` from the write callback). The Google provider (and the others) go
through this shared client.

## Decision

Introduce a **first-class, cooperative cancellation primitive** in `webfiori/ai`.

### 1. A self-contained pub/sub `CancellationToken`

- `CancellationTokenSource::cancel()` is the **publisher**; `CancellationToken`
  exposes `isCancelled()`, `throwIfCancelled()` (throws
  `OperationCancelledException`), and `onCancel(callable)` to **subscribe**.
- Multiple subscribers may register; on `cancel()` **all** fire so different
  layers react independently (HTTP client closes the socket, the tool loop
  stops, the application persists/handles the partial result, loggers record
  it).
- A subscriber registered **after** cancellation has already fired runs
  immediately; a failing subscriber does not prevent the others; `cancel()` is
  idempotent.
- **It is deliberately self-contained — NOT built on `WebFiori/pubsub`.** That
  package is still in progress; coupling a core framework capability to an
  unstable dependency is avoided. A small internal listener list is used
  instead. Migrating to `WebFiori/pubsub` once it stabilises is a documented
  future option, not a v1 dependency.

### 2. Passed via `$options` — no signature changes

The token is passed through the existing `$options` array under a new
`ChatOption::CANCELLATION_TOKEN` constant. This reuses the established
`ChatOption` convention (discoverable, typo-safe) and means **no signature
changes** to `ProviderInterface::chat()` / `streamChat()` or
`HttpClientInterface`. Every provider reads
`$options[ChatOption::CANCELLATION_TOKEN] ?? null`.

#### How the token reaches the transport

`HttpClientInterface::send()` / `sendStreaming()` take only an `HttpRequest`
(no options bag), and there are several `HttpClientInterface` implementations
(cURL plus the Retryable / RateLimitAware / Recording / Replay decorators). To
avoid changing any of those signatures (per ADR-0029), the token travels **on
the `HttpRequest` value object itself**: an optional 5th constructor parameter
`?CancellationToken $cancellationToken = null` plus a
`getCancellationToken()` accessor, and an immutable
`withCancellationToken(?CancellationToken): self` that returns a copy carrying
the token (preserving the value object's immutability — the original is never
mutated). The decorator clients forward the request unchanged.

Providers build their `HttpRequest` through a variety of delegated helpers
(per-provider formatters, Bedrock invocation strategies, the Google
Interactions builder) behind shared interfaces. Rather than thread a token
parameter through every one of those helpers and their interfaces — large
surface, and a burden every *future* formatter would have to repeat — token
wiring is centralised at the **single dispatch point** in `AbstractClient`:
after calling the provider's `buildChatRequest()` / `buildStreamChatRequest()`,
the shared flow applies
`$request = $request->withCancellationToken($this->cancellationFrom($options))`
before sending (`cancellationFrom()` extracts
`$options[ChatOption::CANCELLATION_TOKEN]`). Every current and future provider
therefore inherits cancellation automatically, with **no formatter/strategy or
interface changes**. A **cross-provider guard test** asserts the token reaches
the sent `HttpRequest` for every provider.

Rejected alternatives: threading an explicit token through all formatters /
strategies and their interfaces (most churn, and ongoing burden for new
formatters); and an explicit token parameter on `send()` (signature churn
across `HttpClientInterface` and all implementations). The immutable wither was
chosen over per-builder construction specifically because the delegated
build architecture makes a single central dispatch-point seam cleaner and more
future-proof than editing every builder.

### 3. Non-streaming abort (blocking `curl_exec`)

`CurlHttpClient::send()` reads `$request->getCancellationToken()`; when present
it sets `CURLOPT_NOPROGRESS = false` and a `CURLOPT_PROGRESSFUNCTION` that
returns non-zero when the token is cancelled — cURL then aborts the transfer and
closes the socket. When no token is present the progress options are left
untouched (zero behaviour change for existing callers). The resulting
`CURLE_ABORTED_BY_CALLBACK` (errno **42**) is mapped — **only when the token is
actually cancelled** — to
`OperationCancelledException` (distinct from a transport `HttpException`); all
other errnos keep current behaviour. `chat()` surfaces
`OperationCancelledException`.

### 4. Streaming abort + partial result

`CurlHttpClient::sendStreaming()` checks the token inside the existing
`CURLOPT_WRITEFUNCTION` (returning `0` aborts — the existing rail) and also sets
`CURLOPT_PROGRESSFUNCTION` so cancellation fires **before the first byte**. On
cancel, `streamChat()` invokes an **`onCancel`** callback with the
**accumulated partial `ChatResponse`** (partial text + tokens-so-far + any tools
already called); it does **not** invoke `onComplete`, and cancellation is **not**
surfaced through `onError`. A cancel before any text yields a partial response
with empty text but valid structure.

The `onCancel` callback is passed via `$options` under a new
`ChatOption::ON_CANCEL` constant — **not** as a new `streamChat()` parameter.
This keeps the `streamChat()` signature frozen (consistent with how the token
itself rides `$options`) and reuses the discoverable, typo-safe `ChatOption`
convention. `streamChat()` already accepts `onToken` / `onComplete` / `onError`
as explicit parameters; `onCancel` deliberately travels with the cancellation
feature's other option (`ChatOption::CANCELLATION_TOKEN`) instead, so the whole
cancellation surface is additive via `$options` and no signature changes are
required anywhere.

### 5. Tool-loop propagation

`AbstractClient`'s auto-execute loop checks the token at three points: at the
**top of each iteration**, **before executing each tool** (inside
`executeTools`, ahead of the per-tool try/catch so the cancellation propagates
rather than being captured as a tool failure), and **after tool execution,
before the next provider round-trip** (so a cancel that lands while a tool is
running stops the loop immediately rather than waiting for — or issuing — the
next HTTP call). The token is also carried on every nested request (via the
`withCancellationToken` wither), so an in-flight provider call during the loop
aborts at the HTTP layer too. A cancellation detected in the loop surfaces as
`OperationCancelledException`, which `chat()` records as a **cancellation** (a
`request.cancelled` metric + `Status::CANCELLED` status + a cancelled audit
entry — not `request.failed`) before rethrowing to the caller.

### 6. Granularity (the guarantee)

Cancellation is **best-effort, near-immediate (~sub-second)**: the request is
aborted and the socket torn down within a few hundred milliseconds of
`cancel()`, bounded by cURL's progress-callback cadence. It is a *real* teardown
that stops provider work and cost — but it is **not** a literally-instantaneous
kill. A hard/immediate guarantee would require a different transport model
(threads/processes or a cancellable async client) and is explicitly out of
scope.

### 7. Outcome semantics (all providers)

- **Non-streaming:** throws `OperationCancelledException`.
- **Streaming:** invokes the `ChatOption::ON_CANCEL` callback with the partial
  `ChatResponse`; never `onComplete`; not an error.
- **Tool loop:** surfaces `OperationCancelledException`.

The primitive lives in `ProviderInterface` / `HttpClientInterface` /
`CurlHttpClient` (shared), so Google/OpenAI/Anthropic/Bedrock/Fallback inherit
it; Google is the priority, the rest verified.

## Alternatives considered

- **Cooperative-only (throw from `onToken`) — rejected.** Does not cover
  initial latency or tool execution and does not tear down the socket or stop
  cost. It is a workaround, not cancellation.
- **Build the pub/sub on `WebFiori/pubsub` — deferred.** The package is still in
  progress; the primitive stays self-contained to avoid coupling a core
  capability to an unstable dependency. Revisit when `WebFiori/pubsub` is
  stable.
- **Explicit `?CancellationToken` parameter on the methods — rejected in favour
  of `$options`.** The `ChatOption` convention already carries request options
  discoverably; using it avoids signature churn across the interface and every
  provider while remaining typed via the constant.
- **Hard/immediate kill — out of scope.** Not achievable on a single blocking
  `curl_exec` without a different transport; the best-effort near-immediate
  teardown is sufficient (it stops work and cost).
- **Surface nothing on stream cancel — rejected.** Returning the accumulated
  partial `ChatResponse` lets callers persist/assess partial work; the partial
  is already in hand at cancel time.

## Consequences

- True cancellation: real socket teardown, tool-loop aware, provider-agnostic.
- Fully additive and backward compatible — new class(es), one `ChatOption`
  constant, and internal HTTP wiring; existing callers are unaffected (token
  optional, absent by default).
- Cancellation is cooperative at the cURL boundary, so it is near-immediate, not
  instantaneous — documented as such.
- A small internal pub/sub implementation is maintained in `webfiori/ai` until
  (optionally) migrated to `WebFiori/pubsub`.

## Future considerations

- **An `OnCancelListener` OO contract.** `onCancel()` currently accepts a plain
  `callable`, matching the library's callback convention (`onToken`,
  `onComplete`, metrics callback, etc.) and already supporting object listeners
  via `[$service, 'onCancel']`. A dedicated `OnCancelListener` interface (with
  `onCancel()` accepting `callable|OnCancelListener`, additively) would give an
  explicit, type-hintable OO contract — worthwhile only if the library ships a
  set of reusable, built-in listener objects (e.g. socket-teardown or
  partial-response-persistence listeners). Deferred: not added to the v1
  primitive to avoid two ways of doing the same thing; revisit if #178/#179
  surface a genuine need for shippable listener types. Any addition must remain
  additive so existing closure listeners keep working.

## Tracking

WebFiori/ai milestone **v1.5.0 - Request Cancellation**; Epic **#176**;
sub-issues **#177** (core primitive), **#178** (non-streaming abort), **#179**
(streaming abort + partial), **#180** (tool-loop propagation), **#181** (docs +
example).
