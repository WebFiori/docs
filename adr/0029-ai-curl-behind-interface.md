# ADR-0029: cURL Behind a Custom Interface

**Date:** 2026-07-06
**Status:** Accepted

## Context

The AI library needs to make outbound HTTP requests to provider APIs. The requirements are: POST with JSON bodies, custom headers, streaming responses (SSE), configurable timeouts, and SSL verification. The library also needs to be easily testable without hitting real APIs.

Available options:
1. Use cURL directly (`curl_*` functions) without abstraction
2. Depend on Guzzle or another HTTP library
3. Implement PSR-18 (HTTP Client interface from `psr/http-client`)
4. Use cURL behind a custom `HttpClientInterface`

## Decision

Use cURL as the default transport implementation behind a custom `HttpClientInterface`. The interface defines two methods: `send()` for standard requests and `sendStreaming()` for chunked/SSE responses.

A `CurlHttpClient` is the default implementation. A `FakeHttpClient` is provided for testing, allowing pre-queued responses and request recording.

## Alternatives Considered

**cURL without abstraction:** Cannot be mocked in tests. Provider tests would require network access or complex workarounds.

**Guzzle:** Adds a significant external dependency (Guzzle itself + PSR-7 + PSR-17 + discovery). The library only needs basic HTTP — Guzzle is overkill and contradicts the zero-dependency goal.

**PSR-18 (psr/http-client):** Would add `psr/http-client`, `psr/http-message`, and `psr/http-factory` as dependencies. These are interface-only packages, but they still add external dependencies and require a PSR-7 implementation (another dependency). The streaming use case (`sendStreaming` with chunk callbacks) is also not covered by PSR-18.

**Custom interface with cURL:** Zero external dependencies. Full control over the streaming API. Simple to mock with `FakeHttpClient`. Developers who want PSR-18 compatibility can write a thin adapter.

## Consequences

- Zero external runtime dependencies beyond `ext-curl` (bundled with PHP)
- Streaming is a first-class concern in the interface (not an afterthought)
- Testing is straightforward via `FakeHttpClient` — no network calls needed
- Developers cannot plug in Guzzle or other HTTP clients without writing an adapter
- The library owns its HTTP abstraction — this is a maintenance burden but gives full control over the streaming API
