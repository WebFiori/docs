# ADR-0049: AI: Semantic Response Caching

**Date:** 2026-08-31
**Status:** Proposed

## Context

The library's existing response cache (ADR-0033) is an **exact-match** cache:
it hashes (provider + model + messages + options) with SHA-256 and returns a
stored response only when every byte matches. This works well for deterministic
retries, embeddings, and test replay, but provides near-zero value for
conversational chat — real users almost never send byte-identical requests.

A user asking "What is PHP?" followed by "what's php?" or "explain PHP to me"
will always miss the exact cache, even though the answer is the same. For
applications with high query overlap (FAQ bots, support agents, knowledge-base
assistants), this leaves significant cost savings on the table.

**Semantic caching** matches requests by *meaning* rather than by bytes, using
vector similarity. The library already has the two building blocks this
requires:

1. **Embeddings** — every provider supports `embed()`, producing vectors.
2. **RagProviderInterface** (ADR-0045) — a provider-agnostic contract for
   vector ingest/retrieve with similarity scoring via `RetrievalResult`.

The question is how to layer semantic caching alongside the existing exact-match
cache without compromising the safety guarantees of exact matching.

## Decision

Add a **SemanticCache** component backed by a `RagProviderInterface` instance
and a provider for embeddings. It is a **standalone class**, not an
implementation of `CacheInterface`, because the lookup model is fundamentally
different (text similarity vs. string-key equality).

The two caches form a **tiered lookup** in `AbstractClient::chat()`:

```
1. Policy gate  (enabled? temperature? no auto_execute_tools?)
2. EXACT cache  → CacheInterface::get($key)              [safe, ~free]
3. SEMANTIC     → SemanticCache::lookup($query, $config)  [approximate, costs 1 embed]
4. Call provider
5. Populate both: exact.set() + semantic.store()
```

Exact match is checked first because it is safe (never wrong) and free (hash
comparison). Semantic is checked only on an exact miss. The provider is called
only when both miss.

### SemanticCache API

```php
class SemanticCache {
    public function __construct(
        RagProviderInterface $store,
        ProviderInterface $embedder,
    ) {}

    public function lookup(string $query, SemanticCacheConfig $config): ?ChatResponse {}
    public function store(string $query, ChatResponse $response, SemanticCacheConfig $config): void {}
    public function clear(): void {}
}
```

- `lookup()` embeds the query, calls `$store->retrieve($query, topK: 1)`,
  checks `score >= $config->getThreshold()`, and reconstructs the
  `ChatResponse` from the result's metadata.
- `store()` serializes the `ChatResponse` via `toArray()` and ingests it
  via `$store->ingest($query, ['cached_response' => ..., ...])`.

### Response Serialization (Prerequisite)

`ChatResponse` and its composed value objects (`Message`, `Usage`,
`ContentPart`, `ToolCall`, `ToolResult`) currently have **no** explicit array
or JSON serialization. The existing exact cache stores the live PHP object in
`CachedResponse::$data` (typed `mixed`), so it never crosses a serialization
boundary.

A RAG-backed semantic cache must persist the response in
`RagProviderInterface::ingest()`'s metadata, which vector stores serialize
(typically as JSON). This requires adding `toArray()` and static `fromArray()`
methods to the response object graph.

This is implemented as a separate, prerequisite piece of work because it is
independently valuable (structured export/import of responses for logging,
debugging, and interoperability).

### Configuration

```php
class SemanticCacheConfig {
    public function __construct(
        bool $enabled = false,              // Off by default
        float $threshold = 0.95,            // Cosine similarity threshold
        int $ttlSeconds = 3600,             // TTL for stored entries
        bool $singleTurnOnly = true,        // Only cache single-turn (1 user message)
    ) {}
}
```

- **`enabled` defaults to `false`** — semantic caching is opt-in. This is the
  most important safety default: approximate matching can return a wrong answer
  (false hit), so the user must explicitly accept that tradeoff.
- **`threshold` defaults to 0.95** — conservative, favoring precision over
  recall. Users can lower it to increase hit rate at the cost of accuracy.
- **`singleTurnOnly` defaults to `true`** — multi-turn conversations are
  context-dependent ("give an example" means different things depending on
  prior turns). Single-turn is the safe starting point.

### What Gets Embedded

When `singleTurnOnly` is `true`, only the last user message's text content is
embedded. This is simple, predictable, and avoids the hard problem of
representing multi-turn context as a single vector.

A future pluggable **embedding strategy** can support multi-turn by embedding
a conversation summary or a concatenation of recent turns. This is explicitly
deferred.

### Observability

The existing `Status` enum and metrics pipeline are extended:

- `Status::SEMANTIC_CACHE_HIT` / `Status::SEMANTIC_CACHE_MISS` events.
- `semantic_cache.hit` / `semantic_cache.miss` metrics, with similarity
  `score` attached to hits so operators can monitor match quality and tune
  the threshold.

## Architecture

```
AbstractClient::chat()
  │
  ├─ [exact]    CacheInterface ──→ CacheKeyGenerator::forChat()
  │             InMemoryCache, FileCache, RedisAdapter, ...
  │
  └─ [semantic] SemanticCache ──→ RagProviderInterface::retrieve()
                │                  └─ RetrievalResult (score + metadata)
                │
                ├─ RagProviderInterface::ingest()  (store)
                │
                └─ ProviderInterface::embed()      (query embedding)

Config:
  CacheConfig           → exact cache policy (ADR-0033)
  SemanticCacheConfig   → semantic cache policy (this ADR)
```

## Alternatives Considered

### Semantic cache as a CacheInterface implementation

`CacheInterface::get(string $key)` receives a pre-hashed key — the original
query text is lost. A semantic cache cannot recover it. We would need to change
the interface signature, breaking all existing implementations.

Rejected: the interfaces serve fundamentally different lookup models. Forcing
them together creates a leaky abstraction.

### Replace exact cache with semantic cache

Semantic lookup always costs an embedding API call + vector search. Exact
lookup is a hash comparison (~free). For the cases where exact match works
(retries, embeddings, temperature=0 deterministic calls), replacing it with
semantic would be slower and more expensive with no benefit.

Rejected: the two caches complement each other; exact is the fast safe path,
semantic catches the misses that matter.

### Use PHP native serialize() for response storage

`serialize($chatResponse)` would avoid adding explicit `toArray()`/`fromArray()`
methods. However, it produces opaque binary blobs that break on class
refactoring, are not portable to non-PHP systems, and may exceed metadata size
limits in managed vector stores.

Rejected: explicit array serialization is durable, portable, and inspectable.

### Embed the full conversation for multi-turn

Concatenating all messages and embedding the full text is simple but produces
poor embeddings for long conversations (diluted signal) and uses excessive
tokens. Summarization-based embedding is better but requires an LLM call per
cache check, which is expensive.

Deferred: start with single-turn only (`singleTurnOnly: true`), add pluggable
multi-turn strategy as follow-up once single-turn is validated.

## Consequences

### Easier

- FAQ bots and knowledge-base assistants see cache hits for reworded questions
  without any user code — just enable the config.
- Cost reduction for high-overlap workloads (support, onboarding, repetitive
  analysis) via fewer provider API calls.
- Reuses existing `RagProviderInterface` (ADR-0045) — any RAG backend
  (local, Google, Bedrock, Pinecone) works as the semantic cache store.
- `ChatResponse::toArray()`/`fromArray()` is independently useful for logging,
  debugging, export, and integration testing.
- Exact-match cache remains untouched and safe for its use cases.
- Default-off means no behavior change for existing users.

### Harder

- **False hits are the core risk.** Semantically similar but contextually
  different questions ("enable X" vs. "disable X") can match above the
  threshold. The 0.95 default and single-turn-only scoping mitigate this, but
  users must understand the tradeoff.
- Every semantic cache lookup costs one embedding API call. For high-traffic
  applications, this cost should be weighed against the savings from avoided
  chat completions.
- Response serialization (`toArray()`/`fromArray()`) adds surface area across
  6 classes that must stay in sync as the object model evolves.
- Multi-turn semantic caching remains unsolved; single-turn is a subset of
  the real problem.
