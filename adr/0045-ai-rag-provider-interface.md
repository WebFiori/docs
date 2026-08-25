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

Add `RagProviderInterface` as the single developer-facing contract for
retrieval-augmented generation, regardless of where the underlying index lives.

`RetrieverInterface` is **removed** — `RagProviderInterface` replaces it entirely.
`VectorStorageInterface` is **internal** to `LocalRagProvider` only and is not
exposed to consumers. The `Retriever` class still exists and implements
`RagProviderInterface` for backward compatibility, but `LocalRagProvider` is the
preferred entry point for new code.

```php
interface RagProviderInterface {
    /** @return RetrievalResult[] */
    public function retrieve(string $query, int $topK = 5, array $options = []): array;
    public function ingest(string $content, array $metadata = []): string;
    public function delete(string $id): void;
}
```

## Design Decisions

### Single Interface

`RagProviderInterface` is the only contract consumers interact with. Both
`AgentMemory` and `RetrievalTool` accept `RagProviderInterface` directly — not
`VectorStorageInterface` or `RetrieverInterface`.

### Auth Extraction

`Auth/GoogleAuth` and `Auth/AwsSigner` + `Auth/AwsCredentialChain` are reusable
auth utilities extracted from provider-specific code.

- **GoogleAuth** supports Application Default Credentials (ADC): environment
  variable → gcloud CLI → GCE metadata server.
- **AwsSigner** handles SigV4 signing for any AWS service.

These are shared across all providers that need Google or AWS authentication,
avoiding duplicated auth logic in each provider implementation.

## Architecture

```
Developer-facing:
  RagProviderInterface (retrieve / ingest / delete)
    ├── LocalRagProvider(VectorStorageInterface, ProviderInterface)
    ├── GoogleRagProvider(GoogleRagConfig) → uses Auth/GoogleAuth
    └── BedrockKnowledgeBaseProvider(BedrockKbConfig) → uses Auth/AwsSigner

Backward compatibility:
  Retriever — implements RagProviderInterface (deprecated, use LocalRagProvider)

Consumers accept RagProviderInterface:
  AgentMemory(RagProviderInterface)
  RetrievalTool(RagProviderInterface)

Internal (not developer-facing):
  VectorStorageInterface — pluggable backend for LocalRagProvider
  Auth/GoogleAuth — reusable Google ADC + service account auth
  Auth/AwsSigner — reusable AWS SigV4 signing
```

## Implementations

**LocalRagProvider(VectorStorageInterface, ProviderInterface)** — local vector stores.
Wraps an embedder and a vector store backend (FileVectorStore, SqliteVectorStore, etc.).

```php
$rag = new LocalRagProvider(
    store: new FileVectorStore('/path/to/store'),
    embedder: $openaiClient,
);
```

**GoogleRagProvider(GoogleRagConfig)** — calls Google RAG API directly. Uses
`Auth/GoogleAuth` for authentication, supporting ADC (environment variable,
gcloud CLI, or metadata server). Note: `ingest()` throws
`UnsupportedFeatureException` — Google RAG corpora are managed externally.

```php
$rag = new GoogleRagProvider(new GoogleRagConfig(
    projectId: 'my-project',
    location: 'us-central1',
    corpusId: 'my-corpus',
    credentials: null, // string path, array service account, or null for ADC
));
```

**BedrockKnowledgeBaseProvider(BedrockKbConfig)** — calls AWS Bedrock Knowledge
Bases. Uses `Auth/AwsSigner` for SigV4 authentication. Note: `ingest()` and
`delete()` throw `UnsupportedFeatureException` because Bedrock Knowledge Bases
use S3 data source sync rather than per-document ingestion.

```php
$rag = new BedrockKnowledgeBaseProvider(new BedrockKbConfig(
    region: 'us-east-1',
    knowledgeBaseId: 'KB123',
));
```

## Integration

`RagProviderInterface` is the single injection point wherever RAG is used:

```php
// Single injection point everywhere:
$tool = new RetrievalTool($ragProvider);
$memory = new AgentMemory($ragProvider);
```

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

**Keep RetrieverInterface alongside RagProviderInterface:**
Two interfaces for the same purpose confuses developers. Having both creates
ambiguity about which to implement and which to depend on. Rejected — consolidated
into `RagProviderInterface` as the single contract.

**Embed auth in each provider:**
Duplicates logic across GoogleRagProvider, BedrockKnowledgeBaseProvider, and
any future cloud providers. Makes auth untestable in isolation and forces
credential handling code to be repeated. Rejected — extracted to `Auth/` namespace
as reusable utilities.

## Consequences

**Easier:**
- Any managed RAG service integrates with zero custom code
- `AgentMemory` works with GCP, AWS, Azure, or local store transparently
- New RAG providers (Pinecone, Weaviate, etc.) follow one clear pattern
- `RetrievalTool` and `AgentMemory` have a single injection point
- Auth logic is reusable across all providers needing Google/AWS credentials
- No interface proliferation — one contract to learn and implement

**Harder:**
- `ingest()` semantics differ between providers (local = embed+store, managed = upload document)
- Some managed services don't support programmatic ingestion (Bedrock KB throws `UnsupportedFeatureException`)
- Migration from `Retriever`/`RetrieverInterface` to `RagProviderInterface` required
