# ADR-0031: Logging via Callback Instead of PSR-3

**Date:** 2026-07-06
**Status:** Accepted

## Context

The AI library needs to provide logging for requests, responses, errors, and operational events (token usage, latency). Developers need this for debugging and cost tracking.

Options:
1. Depend on PSR-3 (`psr/log`) and accept a `LoggerInterface`
2. Use a simple callback function
3. No logging support (leave entirely to the developer)

## Decision

Use a simple callable as the logging mechanism, provided via a `LoggerTrait`:

```php
$provider->setLogCallback(function (string $level, string $message, array $context) {
    // Developer bridges to any logger they want
});
```

Standard levels are used: `debug`, `info`, `warning`, `error`.

If no callback is configured, logging calls are no-ops with negligible overhead (a single null check).

## Alternatives Considered

**PSR-3 (`psr/log`):** Adds an external dependency. While `psr/log` is interface-only and lightweight, it contradicts the zero-dependency design goal. Every PSR interface added is a commitment to maintain compatibility with that standard.

**No logging at all:** AI API calls are expensive and slow. Without logging, developers have no visibility into what the library is doing, how many tokens are consumed, or why requests fail. This is unacceptable for production use.

**Built-in file logger:** Too opinionated. Would impose file paths, rotation policies, and format decisions that belong to the application.

## Consequences

- Zero external dependencies for logging
- Developers have full control over where logs go and how they're formatted
- The same callback pattern is reused for metrics and audit logging (consistency)
- Developers who use PSR-3 write a one-line bridge: `$provider->setLogCallback([$logger, 'log'])`
- No log level filtering in the library — the callback receives all levels and the developer filters as needed
- Logging overhead when disabled is a single null check per call
