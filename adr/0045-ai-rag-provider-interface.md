# ADR-0045: AI: RagProviderInterface — Provider-Agnostic RAG

**Date:** 2026-08-23
**Status:** Accepted

## Context

The current RAG implementation is tightly coupled to our own vector store
abstraction (`VectorStorageInterface`). It requires:
1. A separate embedding step (call provider → get vector)
2. A separate storage step (store vector in FileVectorStore or SqliteVectorStore)
3. A retrieval step (embed query → vector search → return chunks)

This works for self-hosted vector stores but breaks down for managed RAG services
(GCP Vertex AI Search, AWS Bedrock Knowledge Bases, Azure AI Search, Pinecone,
Weaviate, Qdrant) — these services handle embedding internally and never expose
raw vectors. You send a text query, you get back ranked documents.

The same provider-agnostic pattern applied to AI providers in ADR-0028 should
apply to RAG.

## Decision

Add `RagProviderInterface` as the top-level abstraction for retrieval-augmented
generation, regardless of where the underlying index lives.

```php
interface RagProviderInterface {
    /**
     * Retrieves relevant documents for a query.
     */
    public function retrieve(string $query, int $limit = 5, array $options = []): array; // RetrievalResult[]

    /**
     * Ingests content into the RAG store.
     */
    public function ingest(string $content, array $metadata = []): string; // → document ID

    /**
     * Deletes a document by ID.
     */
    public function delete(string $id): void;
}
```

### Implementations

**LocalRagProvider** — wraps the existing `VectorStorageInterface` + embedder.
Backward compatible — existing code using `Retriever` continues to work.

```php
$rag = new LocalRagProvider(
    store: new FileVectorStore('/path/to/store'),
    embedder: $openaiClient,
);
```

**VertexAISearchProvider** — calls Vertex AI Search API directly.

```php
$rag = new VertexAISearchProvider(new VertexAISearchConfig(
    projectId: 'my-project',
    location: 'us-central1',
    dataStoreId: 'my-datastore',
    credentials: '/path/to/key.json',
));
```

**BedrockKnowledgeBaseProvider** — calls AWS Bedrock Knowledge Bases.

```php
$rag = new BedrockKnowledgeBaseProvider(new BedrockKbConfig(
    region: 'us-east-1',
    knowledgeBaseId: 'KB123',
    accessKey: '...',
    secretKey: '...',
));
```

### Integration

`RagProviderInterface` becomes the single injection point wherever RAG is used:

```php
// RetrievalTool accepts RagProviderInterface
$tool = new RetrievalTool($rag);

// AgentMemory accepts RagProviderInterface
$memory = new AgentMemory($rag);

// Retriever wraps RagProviderInterface (or stays as LocalRagProvider internally)
```

### Backward Compatibility

`LocalRagProvider` wraps the existing `VectorStorageInterface` + `Retriever`.
All existing code using `FileVectorStore`, `SqliteVectorStore`, or `Retriever`
continues to work — `LocalRagProvider` is just a new entry point.

## Alternatives Considered

**Extend VectorStorageInterface with text search:**
`VectorStorageInterface` is about raw vectors. Managed services don't expose
vectors at all. Forcing them into this interface would require fake vectors.
Rejected.

**Separate TextSearchInterface:**
Two interfaces for what is conceptually one thing (retrieval) creates confusion.
`RagProviderInterface` unifies both cases. Rejected.

**Keep current architecture, add adapters per service:**
Adapters would need to fake the vector operations (embed query, pretend to store,
etc.). Leaky abstraction. Rejected.

## Consequences

**Easier:**
- Any managed RAG service integrates with zero custom code
- `AgentMemory` works with GCP, AWS, Azure, or local store transparently
- New RAG providers (Pinecone, Weaviate, etc.) follow one clear pattern
- `RetrievalTool` and `AgentMemory` have a single injection point

**Harder:**
- `ingest()` semantics differ between providers (local = embed+store, managed = upload document)
- Some managed services don't support programmatic ingestion via the same API
- `LocalRagProvider` adds a thin wrapper over existing code
