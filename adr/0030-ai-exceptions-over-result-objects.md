# ADR-0030: Exceptions Over Result Objects for Error Handling

**Date:** 2026-07-06
**Status:** Accepted

## Context

The AI library needs an error handling strategy for API failures, authentication errors, rate limits, invalid configuration, and unsupported operations. The two main patterns in PHP are:

1. **Exceptions** — Throw typed exceptions, caller uses try/catch
2. **Result objects** — Return a result wrapper with `isSuccess()`, caller checks before using the value

## Decision

Use a typed exception hierarchy as the primary error handling mechanism. All exceptions extend a common `AiException` base class (which extends `\RuntimeException`), enabling both specific catch blocks and catch-all patterns.

Exception hierarchy:
- `AiException` — base (catch-all)
  - `AuthenticationException` — invalid credentials (401/403)
  - `RateLimitException` — rate limit exceeded (429), carries retry-after
  - `ProviderException` — generic API errors, carries status code and provider error code
  - `InvalidConfigException` — bad configuration, carries option name
  - `HttpException` — transport failures, carries cURL error code
  - `StreamingException` — stream processing errors
  - `UnsupportedFeatureException` — feature not available on provider

Each exception carries contextual data relevant to recovery (retry-after seconds, status codes, error codes).

## Alternatives Considered

**Result objects (`$result->isSuccess()`):** More verbose, easy to forget to check (silent failures). Requires wrapping every return type. Better suited for validation where failure is expected and normal — but AI API calls are meant to succeed.

**Single exception class with error codes:** Less ergonomic. Forces callers to switch on error codes instead of catching specific types. Doesn't carry type-specific context (retry-after only makes sense for rate limits).

## Consequences

- The happy path is clean — no `if ($result->isSuccess())` wrappers
- Unhandled errors fail loudly rather than silently
- Callers can catch at the granularity they need (specific type vs `AiException` catch-all)
- Each exception carries recovery-relevant data (retry-after, status code, option name)
- Matches PHP ecosystem conventions (Guzzle, Doctrine, Symfony all throw exceptions)
- Callers must use try/catch — no way to "ignore" errors without explicit choice
