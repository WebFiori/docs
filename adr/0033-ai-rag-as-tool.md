# ADR-0033: AI: RAG via Tool Calling Instead of Pipeline Wrapping

**Date:** 2026-08-16
**Status:** Accepted

## Context

Retrieval-Augmented Generation (RAG) requires three steps: embed a query, search
a vector store for relevant chunks, and inject the retrieved context into the
prompt before sending it to a chat model.

There are two common integration patterns:

**Pattern A — Pipeline wrapping:**
The RAG pipeline wraps the chat call. On every `chat()`, the pipeline
automatically embeds the last user message, retrieves chunks, and injects them
into the system prompt before forwarding to the model.

**Pattern B — RAG as a tool:**
The retrieval step is exposed as a tool. The chat model decides when to invoke
it. The library provides `RetrieverInterface`, `Retriever`, and `RetrievalTool`.
The developer registers the tool and calls `chat()` normally.

## Decision

Implement RAG as a tool (Pattern B), not as a pipeline wrapper (Pattern A).

The `RetrievalTool` implements `ToolInterface` and wraps a `RetrieverInterface`.
The model invokes it when it determines that a knowledge lookup is needed:

```php
$retriever = new Retriever($embedProvider, $vectorStore);
$ragTool   = new RetrievalTool($retriever, name: 'search_knowledge');

$response = $chatProvider->chat(
    messages: $messages,
    options: [
        'tools'              => [$ragTool],
        'auto_execute_tools' => true,
    ],
);
```

The two providers are completely independent:

```
User question
     │
     ▼
chat model (e.g. gemini-2.5-flash)
     │  decides to call search_knowledge("water withdrawal")
     ▼
RetrievalTool::execute()
     ├── embedding model embeds the query
     ├── VectorStore searches by cosine similarity
     └── returns JSON with retrieved text chunks
     │
     ▼
chat model reads retrieved chunks, formulates final answer
```

The embedding model is only invoked inside the tool. The chat model never
sees or handles vectors — it only receives the tool result as structured JSON.

## Alternatives Considered

**Pattern A — Automatic pipeline wrapping:**

```php
$pipeline = new RagPipeline($chatProvider, $embedProvider, $vectorStore);
$response = $pipeline->chat($messages); // Always retrieves
```

Rejected because:
- Retrieval happens on every message, including greetings and simple questions
  where it wastes latency and API calls
- The developer has no visibility into when retrieval occurred
- Harder to debug — the prompt modification is invisible
- Breaks composability with other tools

**RAG as a `chat()` option:**

```php
$response = $chatProvider->chat($messages, options: ['rag' => $retriever]);
```

Rejected because it leaks RAG concerns into the core `AbstractClient`, making
it a special case rather than a composable feature.

## Consequences

**Easier:**
- Model autonomy — the model only retrieves when it determines it is needed,
  avoiding unnecessary latency for simple questions
- Composable — the retrieval tool sits alongside other tools (web search,
  calculator, database queries) with no special treatment
- Transparent — tool calls are visible in `ChatResponse::getMessage()->getToolCalls()`
- Testable — `RetrieverInterface` can be mocked without touching the provider
- The embedding and chat models are fully decoupled; either can be swapped
  independently

**Harder:**
- Always-on RAG (inject context on every message) requires the developer to
  call `retrieve()` manually and inject into the system message themselves
- The model may sometimes skip retrieval when it should have searched; prompt
  engineering (system message instructions) is needed to guide this behavior
