# ADR-0027: AI Library as Standalone Package

**Date:** 2026-07-06
**Status:** Accepted

## Context

WebFiori needs an AI library to support chat completions, embeddings, image generation, tool calling, and streaming across multiple providers (OpenAI, Google Vertex AI, Anthropic, AWS Bedrock).

The question was whether this library should be:
1. Tightly coupled to the WebFiori framework (depending on its routing, middleware, services)
2. A standalone Composer package usable independently
3. Both (core package + framework integration layer)

Additionally, we considered whether to include:
- A built-in RAG (Retrieval-Augmented Generation) pipeline
- AI-powered middleware (content moderation, summarization, spam detection)

## Decision

The AI library is a **standalone Composer package** (`webfiori/ai`) with:
- Namespace: `WebFiori\Ai`
- No dependency on the WebFiori framework or any of its packages
- No built-in RAG pipeline — provide building blocks (embeddings + vector storage + LLM) and let developers compose them
- No AI-powered middleware — that belongs in the application layer

The library depends only on PHP >=8.1, ext-curl, and ext-json.

## Alternatives Considered

**Tight framework coupling:** Would limit adoption to WebFiori users only. AI functionality is generic and should not be locked to a specific framework's routing or service layer.

**Both (core + integration layer):** Adds complexity with two packages to maintain. The framework integration (registering providers as services, AI middleware) can be done by the developer in a few lines without a dedicated package.

**Built-in RAG pipeline:** RAG involves opinionated decisions about chunking strategies, prompt templates, and retrieval configuration. These vary significantly per use case. Providing the building blocks (embeddings API, vector storage interface, chat completions) gives developers full control without imposing a specific RAG architecture.

**AI-powered middleware:** Content moderation, summarization, and spam detection are application-level policies, not library concerns. Different applications have different requirements for what to moderate and how to respond.

## Consequences

- The library can be used in any PHP project, not just WebFiori applications
- Developers who want RAG must compose the pipeline themselves using the provided building blocks
- No framework-specific convenience features (auto-wiring, route integration) — those are left to the developer or a future optional integration package
- The library has zero external runtime dependencies beyond PHP extensions, keeping it lightweight
