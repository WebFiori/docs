# ADR-0046: AI: AgentMemory — Persistent Learning for Agents and Clients

**Date:** 2026-08-25
**Status:** Accepted

## Context

Agents and chat clients benefit from persistent memory. A user may correct an
agent or provide important context that should persist across sessions. Without
memory, each conversation starts from zero — corrections are lost, preferences
forgotten, and domain knowledge must be re-explained.

AgentMemory adds RAG-based persistent knowledge that agents and clients
automatically recall and learn from. It leverages existing embedding and vector
storage infrastructure to store facts, retrieve them by semantic similarity,
and inject relevant knowledge into prompts before generation.

## Decision

Add `AgentMemory` class and `RememberStrategyInterface` to enable persistent
learning. Memory integrates at both the `AgentTool` and `AbstractClient` levels
with a recall-before, remember-after pattern.

### Design Decisions

**1. AgentMemory wraps RagProviderInterface**

Reuses existing RAG infrastructure (`LocalRagProvider`, `GoogleRagProvider`,
`BedrockKnowledgeBaseProvider`). No new storage abstraction needed.

```php
use WebFiori\Ai\Tool\AgentMemory;
use WebFiori\Ai\Rag\LocalRagProvider;
use WebFiori\Ai\Embedding\SqliteVectorStore;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;

$ragProvider = new LocalRagProvider(
    store: new SqliteVectorStore('/path/to/memory.db'),
    embedder: new OpenAIClient(new OpenAIClientConfig(
        apiKey: 'sk-...',
        model: 'text-embedding-3-small',
    )),
);

$memory = new AgentMemory(
    ragProvider: $ragProvider,
    minScore: 0.75,
    topK: 5,
);
```

**2. RememberStrategyInterface decouples fact extraction**

Strategies determine WHEN and WHAT to remember. The interface is minimal:

```php
interface RememberStrategyInterface {
    /**
     * @param Message[] $messages The conversation messages.
     * @param string $agentResponse The agent's/model's response.
     * @return string[] Facts to store (empty array if nothing to remember).
     */
    public function extract(array $messages, string $agentResponse): array;
}
```

Three built-in strategies:

`ManualRememberStrategy` — no-op, user calls `remember()` explicitly:

```php
use WebFiori\Ai\Tool\ManualRememberStrategy;

$strategy = new ManualRememberStrategy();
$strategy->extract($messages, $response); // always returns []
```

`KeywordRememberStrategy` — regex-based correction detection (free, zero API cost):

```php
use WebFiori\Ai\Tool\KeywordRememberStrategy;

// Default patterns: "actually", "correction:", "wrong", "remember that", "fyi", etc.
$strategy = new KeywordRememberStrategy();

// Custom patterns:
$strategy = new KeywordRememberStrategy([
    '/\bprefer\b/i',
    '/\balways use\b/i',
    '/\bnever\b/i',
]);
```

Default patterns detect 15 common correction indicators:

| Pattern | Example |
|---------|---------|
| `actually` | "Actually, my name is spelled Ibrahim" |
| `correction:` | "Correction: the port is 5432" |
| `no, it's` / `no, it is` | "No, it's PostgreSQL not MySQL" |
| `wrong` / `incorrect` | "That's wrong, the version is 3.1" |
| `instead, use` | "Instead, use the v2 API" |
| `the correct` | "The correct endpoint is /api/v2" |
| `remember that` | "Remember that I use dark mode" |
| `important:` / `fyi` | "FYI, deployments are on Fridays" |

`LLMRememberStrategy` — uses a classifier LLM to extract facts (accurate, costs
1 API call per exchange):

```php
use WebFiori\Ai\Tool\LLMRememberStrategy;
use WebFiori\Ai\Provider\Google\GoogleClient;

$classifier = new GoogleClient(new GoogleClientConfig(
    apiKey: '...',
    model: 'gemini-2.5-flash',
));

$strategy = new LLMRememberStrategy(
    classifier: $classifier,
    model: 'gemini-2.5-flash', // optional override
);

// Internally sends:
// "Analyze this conversation exchange and extract any corrections,
//  new facts, or important information..."
// Returns JSON array of fact strings, parsed automatically.
```

**3. Two integration levels**

Works on both `AgentTool` (specialist agents) and `AbstractClient` (any provider).
Same `AgentMemory` instance can be shared across multiple clients:

```php
// Shared memory
$memory = new AgentMemory($ragProvider);

// Attach to a client
$client->setMemory($memory);
$client->setRememberStrategy(new KeywordRememberStrategy());

// Same memory attached to an agent tool
$agent->setMemory($memory);
```

**4. Recall before, remember after**

In `execute()`/`chat()`:
- Before: recall relevant memories, inject into system prompt as `## Relevant Knowledge`
- After: run `RememberStrategy` to extract facts, store via `remember()`

```php
// What happens inside chat():
$results = $memory->recall('user query text');
// Internally: $this->ragProvider->retrieve($query, $topK), filtered by minScore

// Inject into system prompt:
// ## Relevant Knowledge
// - The user prefers dark mode (score: 0.92)
// - Deployments happen on Fridays (score: 0.85)

$response = $this->sendRequest($messages);

// After response:
$facts = $strategy->extract($messages, $response->getMessage()->getContent());

foreach ($facts as $fact) {
    $memory->remember($fact);
    // Internally: $this->ragProvider->ingest($fact, $metadata)
}
```

**5. Supersedes pattern for corrections**

`remember()` accepts optional `$supersedes` ID to delete old memory when a
correction arrives. Prevents contradictions:

```php
// Original memory
$id = $memory->remember('The user prefers light mode.');

// User corrects: "Actually, I prefer dark mode"
$newId = $memory->remember(
    'The user prefers dark mode.',
    metadata: ['source' => 'user_correction'],
    supersedes: $id, // deletes old memory before storing
);
```

**6. Timestamp metadata**

Every memory gets automatic timestamp. Enables future relevance decay or TTL
without breaking changes:

```php
$id = $memory->remember('Deploy target is us-east-1');
// Stored metadata: { "timestamp": 1724601600, "text": "Deploy target is us-east-1" }

// Custom metadata merged with automatic fields:
$memory->remember('Use PHP 8.3', metadata: ['confidence' => 'high']);
// Stored: { "timestamp": 1724601600, "text": "Use PHP 8.3", "confidence": "high" }
```

**7. Uses RagProviderInterface directly**

AgentMemory accepts `RagProviderInterface`, working transparently with
`LocalRagProvider` for local vector stores or `GoogleRagProvider` /
`BedrockKnowledgeBaseProvider` for managed cloud RAG services:

```php
// Local vector store:
$memory = new AgentMemory(new LocalRagProvider($vectorStore, $embedder));

// Google RAG corpus:
$memory = new AgentMemory(new GoogleRagProvider(new GoogleRagConfig(
    projectId: 'my-project',
    location: 'us-central1',
    corpusId: 'memory-corpus',
)));

// AWS Bedrock Knowledge Base:
$memory = new AgentMemory(new BedrockKnowledgeBaseProvider(new BedrockKbConfig(
    region: 'us-east-1',
    knowledgeBaseId: 'KB123',
)));
```

## Alternatives Considered

**Always-on auto-remember:**
Every response stores something. Memory fills with noise quickly, making recall
less relevant over time. Rejected.

**Agent self-classifies corrections:**
Agent decides what to remember during generation. Pollutes the model's primary
task, unreliable across providers. Rejected.

**ManagedCorpusInterface for cloud:**
Separate interface for GCP/AWS managed vector services. Rejected — covered by
`RagProviderInterface` which `AgentMemory` now accepts directly.

**Memory in constructor only (no setter):**
Can't share memory across clients or set after construction. Setters provide
flexibility for dependency injection and shared memory scenarios. Rejected.

**Store full conversation in memory:**
Expensive — embeddings for entire exchanges consume storage and degrade recall
precision. Strategy-based extraction is selective and stores only the important
facts. Rejected.

## Consequences

**Easier:**
- Agents learn from corrections across sessions
- Zero-config with `KeywordRememberStrategy` (no API cost)
- Works with existing vector stores, no new dependencies
- Shared memory across agents/clients enables knowledge transfer
- Composable with all existing features (`FallbackProvider`, `ModelRouter`, etc.)

**Harder:**
- Memory growth needs manual management (`forget()` or `supersedes`)
- `KeywordRememberStrategy` may miss subtle corrections
- `LLMRememberStrategy` adds latency and cost per response
- Contradicting memories possible without `supersedes` discipline
