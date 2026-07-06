# ADR-0028: Provider-Agnostic Interface Design

**Date:** 2026-07-06
**Status:** Accepted

## Context

The AI library needs to support multiple providers (OpenAI, Google Vertex AI, Anthropic, AWS Bedrock) that have significantly different API formats, authentication mechanisms, and feature sets. Developers should be able to swap providers without changing application code.

Key differences between providers:
- Authentication: API keys (OpenAI, Anthropic), OAuth2/service accounts (Vertex AI), SigV4 (Bedrock)
- Request formats: Different JSON structures for the same operations
- Feature support: Not all providers support all operations (e.g., Anthropic has no image generation)
- Streaming formats: Different SSE structures across providers

## Decision

Define a single `ProviderInterface` with methods for all supported operations: `chat()`, `embed()`, `generateImage()`, and `streamChat()`. Providers that do not support a specific operation throw `UnsupportedFeatureException`.

The interface uses library-defined DTOs (`Message`, `ChatResponse`, `EmbeddingResponse`, `ImageRequest`, `ImageResponse`) rather than provider-specific formats. Each provider implementation is responsible for mapping between these DTOs and its native API format.

An `HttpClientInterface` abstracts the transport layer, allowing providers to share HTTP infrastructure while enabling test mocking.

Authentication is handled internally by each provider implementation — the interface does not prescribe an auth strategy.

## Alternatives Considered

**Separate interfaces per feature (ChatProviderInterface, EmbeddingProviderInterface, etc.):** More granular, but forces consumers to type-hint against specific capabilities. A single interface with `UnsupportedFeatureException` is simpler for the common case where developers use one provider for all operations.

**Pass-through to native API format (no DTOs):** Would avoid mapping overhead but defeat the purpose of provider-agnosticism. Developers would need to learn each provider's format.

**Abstract base class instead of interface:** PHP interfaces allow multiple implementation inheritance and are easier to mock. A base class would be too prescriptive about internal implementation.

## Consequences

- Developers write application code against `ProviderInterface` and swap providers via configuration
- Each provider implementation must handle its own request/response serialization
- Features unsupported by a provider fail explicitly with a typed exception rather than silently
- The `$options` array parameter provides an escape hatch for provider-specific parameters without bloating the interface
- Adding a new provider requires implementing one interface — no framework changes needed
