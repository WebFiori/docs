# Introduction to WebFiori AI

WebFiori AI (`webfiori/ai`) is a provider-agnostic AI library for PHP. It gives you a single, consistent interface for chat completions, streaming, embeddings, image generation, tool calling, and Retrieval-Augmented Generation (RAG) across multiple providers — OpenAI, Google (Gemini / Vertex AI), Anthropic, and AWS Bedrock.

<meta name="description" content="Overview of the WebFiori AI library: a provider-agnostic PHP interface for chat, streaming, embeddings, images, tools, and RAG across OpenAI, Google, Anthropic, and AWS Bedrock.">

> **Note.** `webfiori/ai` is a standalone library and is **not** a core part of the WebFiori framework. It does not ship with the framework — you must install it explicitly with Composer (see [Installation](#installation)). It has no dependency on the framework and can be used in any PHP project. The library follows its own semantic versioning, independent of the framework version this site otherwise documents.

## Why WebFiori AI

- **Provider-agnostic** — the same code works across OpenAI, Google, Anthropic, and Bedrock. Switching provider is a configuration change, not a rewrite.
- **Standalone** — the library has no dependency on the WebFiori framework; use it in any PHP project.
- **Complete feature set** — chat, token-by-token streaming, embeddings with a built-in vector store, image generation, tool/function calling, and RAG.
- **Production-ready** — retry logic, provider fallback with a circuit breaker, rate-limit awareness, caching, health checks, metrics, PII redaction, and audit logging.

## Requirements

- PHP 8.1 or later
- The `curl` and `json` extensions

## Installation

`webfiori/ai` is a separate package — it is not bundled with the WebFiori framework. Add it to your project explicitly:

```bash
composer require webfiori/ai
```

This works in any PHP project, whether or not you use the WebFiori framework.

## A First Request

```php
<?php

require 'vendor/autoload.php';

use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Message;

$client = new OpenAIClient(new OpenAIClientConfig(
    apiKey: 'sk-...',
    model: 'gpt-4o',
));

$response = $client->chat([
    new Message('system', 'You are a helpful assistant.'),
    new Message('user', 'What is PHP?'),
]);

echo $response->getMessage()->getContent();
```

Every provider client implements the same `ProviderInterface`, so the code above changes only in its configuration when you switch providers.

## Core Concepts

- **Provider client** — one class per provider (`OpenAIClient`, `GoogleClient`, `AnthropicClient`, `BedrockClient`). Each is constructed with a typed config object.
- **`Message`** — a single conversation turn with a role (`system`, `user`, `assistant`, `tool`) and content. Content can be plain text or multi-modal parts (text, images, documents).
- **`ChatResponse`** — the result of a chat call, exposing the assistant `Message`, token `Usage`, finish reason, and request ID.
- **Options array** — per-call settings such as `model`, `temperature`, `max_tokens`, and `tools`, passed as the second argument to `chat()`.

## Feature Support by Provider

Not every operation is available on every provider. Chat, streaming, tool calling, structured output, and vision are supported everywhere; embeddings and image generation are provider-specific. Calling an unsupported operation throws `UnsupportedFeatureException`. See the [Providers](learn/ai-providers) page for the full matrix.

## Where to Next

- [Basic Chat](learn/ai-basic-chat) — send messages and read responses
- [Providers](learn/ai-providers) — supported providers and the feature matrix
- [Configuration](learn/ai-configuration) — configure each provider client
