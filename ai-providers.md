# AI Providers

WebFiori AI supports four providers through one common interface: OpenAI, Google (Gemini and Vertex AI), Anthropic, and AWS Bedrock. This page covers which provider to reach for and which operations each supports.

<meta name="description" content="Supported WebFiori AI providers (OpenAI, Google, Anthropic, AWS Bedrock) and the feature matrix showing which operations each supports.">

## The Provider Clients

| Provider | Client class |
|----------|--------------|
| OpenAI | `WebFiori\Ai\Provider\OpenAI\OpenAIClient` |
| Google (Gemini / Vertex AI) | `WebFiori\Ai\Provider\Google\GoogleClient` |
| Anthropic | `WebFiori\Ai\Provider\Anthropic\AnthropicClient` |
| AWS Bedrock | `WebFiori\Ai\Provider\Bedrock\BedrockClient` |

Each client implements `WebFiori\Ai\Provider\ProviderInterface`, so application code depends on the interface, not the concrete provider:

```php
use WebFiori\Ai\Provider\ProviderInterface;
use WebFiori\Ai\Message;

function summarize(ProviderInterface $ai, string $text): string {
    return $ai->chat([
        new Message('system', 'Summarize the user text in one sentence.'),
        new Message('user', $text),
    ])->getMessage()->getContent();
}
```

You can pass any provider client (or the fallback/router decorators) to that function unchanged.

## Feature Matrix

Chat, streaming, tool calling, structured output, and vision are available on every provider. Embeddings and image generation are provider-specific.

| Feature                    | OpenAI | Google | Anthropic | AWS Bedrock |
|----------------------------|:------:|:------:|:---------:|:-----------:|
| Chat completions           |   ✅   |   ✅   |    ✅     |     ✅      |
| Streaming                  |   ✅   |   ✅   |    ✅     |     ✅      |
| Tool / function calling    |   ✅   |   ✅   |    ✅     |     ✅      |
| Structured output (JSON)   |   ✅   |   ✅   |    ✅     |     ✅      |
| Vision / multi-modal input |   ✅   |   ✅   |    ✅     |     ✅      |
| Embeddings                 |   ✅   |   ✅   |    ❌     |     ❌¹     |
| Image generation           |   ✅   |   ✅   |    ❌     |     ❌¹     |

¹ Not implemented for Bedrock. Anthropic has no embeddings or image-generation API, so use OpenAI or Google for those.

## Unsupported Operations

Calling an operation a provider does not support throws `UnsupportedFeatureException`. The exception carries the feature and provider names so you can route around it:

```php
use WebFiori\Ai\Exception\UnsupportedFeatureException;

try {
    $vectors = $anthropic->embed('some text');
} catch (UnsupportedFeatureException $e) {
    echo $e->getFeature();       // 'embeddings'
    echo $e->getProviderName();  // 'anthropic'
    // fall back to OpenAI or Google for embeddings
}
```

## Switching Providers

Because every client shares the interface, switching provider is only a construction change:

```php
// OpenAI
$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...', model: 'gpt-4o'));

// Google (Gemini)
$client = new GoogleClient(new GoogleClientConfig(apiKey: 'your-gemini-key', model: 'gemini-2.5-flash'));

// Anthropic
$client = new AnthropicClient(new AnthropicClientConfig(apiKey: 'sk-ant-...', model: 'claude-sonnet-4-20250514'));

// AWS Bedrock
$client = new BedrockClient(new BedrockClientConfig(region: 'us-east-1', accessKey: '...', secretKey: '...'));
```

## Self-Hosted and OpenAI-Compatible Endpoints

Because `OpenAIClient` builds every request URL from its configurable `baseUrl`, you can point it at any OpenAI-compatible server. That includes a locally hosted open-source model (Ollama, LM Studio, vLLM, llama.cpp, LocalAI):

```php
$client = new OpenAIClient(new OpenAIClientConfig(
    apiKey: 'local',                       // most local servers ignore this; any non-empty value works
    model: 'llama3.1',
    baseUrl: 'http://localhost:11434/v1',  // e.g. Ollama's OpenAI-compatible API
));
```

Feature availability then depends on what your local server implements (chat and streaming are the most widely supported).

## Where to Next

See [Configuration](learn/ai-configuration) for the full configuration reference for each provider, or [Basic Chat](learn/ai-basic-chat) to start sending messages.
