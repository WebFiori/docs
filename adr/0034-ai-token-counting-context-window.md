# ADR-0034: AI Library Token Counting and Context Window Management

**Date:** 2026-08-09
**Status:** Accepted

## Context

Each AI model has a maximum context window (e.g., GPT-4o: 128k tokens, Claude 3: 200k tokens). When conversations grow long or include many tools, requests can exceed these limits, resulting in API errors. Developers need:

1. A way to estimate token usage before sending requests
2. Automatic handling of context overflow to prevent errors

## Decision

### Token Counting

Use simple character-ratio estimation (~4 characters ≈ 1 token) rather than exact tokenization:

```php
$count = $provider->countTokens($messages);
$count = $provider->countTokens($messages, $tools); // include tool definitions
```

This approach is:
- Zero dependencies (no tiktoken port needed)
- Fast (no HTTP calls to provider APIs)
- Accurate enough for context management (~5-10% margin)

### Context Window Strategy

Auto-truncate in `chat()` and `streamChat()` before sending, with warning logs when truncation occurs:

```php
$provider->setContextWindowStrategy(new SlidingWindowStrategy(
    maxTokens: 128000,
    reserveForCompletion: 4096,
    preserveSystemMessage: true,
));

// Auto-truncates if needed, logs warning
$response = $provider->chat($messages);
```

### Strategy Interface

Define `ContextWindowStrategyInterface` for extensibility:

```php
interface ContextWindowStrategyInterface {
    /**
     * Truncates messages to fit within the token limit.
     *
     * @param Message[] $messages The messages to truncate.
     * @param int $maxTokens Maximum tokens allowed.
     * @param ToolInterface[] $tools Tools that also consume tokens.
     *
     * @return Message[] The truncated messages.
     */
    public function truncate(array $messages, int $maxTokens, array $tools = []): array;
}
```

Built-in implementations:
- `SlidingWindowStrategy` — removes oldest messages first, preserves system message
- `NoTruncationStrategy` — throws exception if limit exceeded

### Configuration

Require manual configuration of context limits (no hardcoded defaults):

```php
$strategy = new SlidingWindowStrategy(maxTokens: 128000);
```

Models change frequently and limits vary by provider tier, fine-tuning, and API version. Explicit configuration ensures accuracy.

### Scope

Context window management applies to:
- `chat()` — auto-truncate before sending
- `streamChat()` — auto-truncate before sending
- `countTokens()` — estimate for messages and optionally tools

## Alternatives Considered

### Exact tokenization (tiktoken)
Adds complexity and dependency. Estimation is sufficient for window management.

### Provider API token counting
Requires HTTP calls, adds latency, not available for all providers.

### Silent truncation without logging
Developers wouldn't know context was lost. Logging provides visibility.

### Hardcoded model limits
Maintenance burden, becomes outdated. Explicit configuration is more reliable.

## Consequences

### Easier
- Developers don't need to manually track context size
- No surprise API errors from context overflow
- Extensible for custom truncation strategies (e.g., summarization)

### Harder
- Developers must configure `maxTokens` for their model
- Estimation may be ~5-10% off from actual token count
- Truncation may silently remove important context (mitigated by logging)
