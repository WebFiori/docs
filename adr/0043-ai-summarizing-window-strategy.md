# ADR-0043: AI: SummarizingWindowStrategy — Summarize Old Context Instead of Truncating

**Date:** 2026-08-23
**Status:** Accepted

## Context

The existing `SlidingWindowStrategy` truncates old messages when the conversation
approaches the model's context limit. This loses information — earlier messages
disappear entirely. For long conversations (chatbots, agents, document analysis),
losing earlier context degrades response quality.

A `SummarizingWindowStrategy` preserves the essence of earlier messages by
summarizing them into a single system message before they fall off, rather than
discarding them silently.

## Decision

Add `SummarizingWindowStrategy` implementing `ContextWindowStrategyInterface`.

### Design Decisions

**1. Injected summarizer provider**

The summarizer is a separate injected `ProviderInterface`. This avoids circular
risk (summarizing a near-full conversation with the same model) and allows using
a cheaper/faster model for summarization.

```php
$strategy = new SummarizingWindowStrategy(
    summarizer: $gpt4oMini,  // cheap model for summaries
    contextWindow: 8192,
    threshold: 0.70,         // summarize at 70% of context window
    keepRecentTurns: 3,      // always keep last 3 user/assistant pairs
);

$client->setContextWindowStrategy($strategy);
// Summary happens automatically when threshold is reached
```

**2. Token threshold trigger (default: 70%)**

Summarization triggers when the estimated token count exceeds a percentage of
the model's context window. Token percentage reflects what actually matters
(context limit) more accurately than a message count.

Default threshold: `0.70` (70% of context window).

**3. Keep system message + last N turns**

When summarization triggers:
- System messages are always preserved verbatim
- The last N user/assistant exchange pairs (turns) are kept verbatim
- Everything between is summarized

Default `keepRecentTurns: 3` (6 messages of fresh context).

**4. Summary injected as system message**

The generated summary is injected as a `Message('system', 'Summary of earlier conversation: ...')`,
positioned immediately after the original system message (if any).

```
[system: "You are a helpful assistant"]          ← original, preserved
[system: "Summary of earlier conversation: ..."] ← injected by strategy
[user: recent message]
[assistant: recent response]
[user: latest message]                           ← current request
```

### Summarization Prompt

```
Summarize the following conversation history concisely.
Preserve key facts, decisions, and context that would be
needed to continue the conversation naturally.

[conversation messages to summarize]
```

### Token Estimation

Uses the existing `TokenEstimator::countMessages()` to estimate token usage
before each chat call. No additional dependencies.

### Caching

The strategy caches the last generated summary in memory. If the same messages
are presented again (e.g., in a tool loop iteration), the cached summary is
reused without another API call.

## Alternatives Considered

**Same client for summarization:**
Circular risk — if the conversation is already at 70% capacity, the summarization
call using the same client could itself trigger another summarization. Rejected.

**Message count threshold:**
A proxy for what actually matters (token count). A conversation of 20 short
messages ≠ 20 long messages. Rejected in favour of token percentage.

**Oldest 50% summarized:**
Unpredictable and not semantically meaningful. "Turns" (user/assistant pairs)
are the natural unit. Rejected.

**Append to existing system message:**
Modifies the original system prompt, which could change the assistant's behavior.
Rejected — a separate system message is cleaner.

## Consequences

**Easier:**
- Long conversations maintain quality without manual history management
- Cheaper summarizer can be used, keeping cost low
- Transparent to callers — same `ProviderInterface`, no code changes needed

**Harder:**
- Extra API call on first summarization event (and after cache invalidation)
- Summary quality depends on the summarizer model and prompt
- Token estimation is approximate — may trigger slightly early or late
