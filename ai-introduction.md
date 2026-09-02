# Introduction to WebFiori AI

WebFiori AI (`webfiori/ai`) is a provider-agnostic AI library for PHP. It gives you one consistent interface for chat completions, streaming, embeddings, image generation, tool calling, and Retrieval-Augmented Generation (RAG). The same code works across OpenAI, Google (Gemini and Vertex AI), Anthropic, and AWS Bedrock.

<meta name="description" content="Overview of the WebFiori AI library: a provider-agnostic PHP interface for chat, streaming, embeddings, images, tools, and RAG across OpenAI, Google, Anthropic, and AWS Bedrock.">

> **Note:** `webfiori/ai` is a standalone library and is **not** a core part of the WebFiori framework. It does not ship with the framework, so you must install it explicitly with Composer (see [Installation](#installation)). It has no dependency on the framework and works in any PHP project. It also follows its own semantic versioning, separate from the framework version this site otherwise documents.

## Why WebFiori AI

It is provider-agnostic, so the same code runs against OpenAI, Google, Anthropic, or Bedrock. Switching provider is a configuration change rather than a rewrite.

It is standalone. There is no dependency on the WebFiori framework, so you can drop it into any PHP project.

It covers the full feature set: chat, token-by-token streaming, embeddings with a built-in vector store, image generation, tool calling, and RAG.

It is built for production, with retry logic, provider fallback backed by a circuit breaker, rate-limit awareness, caching, health checks, metrics, PII redaction, and audit logging.

## Requirements

You need PHP 8.1 or later, plus the `curl` and `json` extensions.

## Installation

Since `webfiori/ai` is a separate package that does not ship with the framework, add it to your project explicitly:

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

Every provider client implements the same `ProviderInterface`, so only the configuration changes when you switch providers.

## Core Concepts

A **provider client** is one class per provider (`OpenAIClient`, `GoogleClient`, `AnthropicClient`, `BedrockClient`), each constructed with a typed config object.

A **`Message`** is a single conversation turn with a role (`system`, `user`, `assistant`, or `tool`) and its content. Content can be plain text or multi-modal parts such as text, images, and documents.

A **`ChatResponse`** is the result of a chat call. It exposes the assistant `Message`, token `Usage`, the finish reason, and the request ID.

The **options array** holds per-call settings such as `model`, `temperature`, `max_tokens`, and `tools`. You pass it as the second argument to `chat()`.

## Feature Support by Provider

Not every operation is available on every provider. Chat, streaming, tool calling, structured output, and vision work everywhere, while embeddings and image generation are provider-specific. Calling an operation a provider does not support throws `UnsupportedFeatureException`. The [Providers](learn/ai-providers) page has the full matrix.

## Where to Next

Start with [Basic Chat](learn/ai-basic-chat) to send messages and read responses. Then see [Providers](learn/ai-providers) for supported providers and the feature matrix, and [Configuration](learn/ai-configuration) to set up each provider client.
