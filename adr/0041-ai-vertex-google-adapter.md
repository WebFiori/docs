# ADR-0041: AI: Vertex AI Model Garden — GoogleAdapter Pattern

**Date:** 2026-08-19
**Status:** Accepted

## Context

Vertex AI hosts third-party models (Anthropic Claude, Meta Llama, Mistral) alongside
Google's own Gemini models. These non-Google models use the same Vertex AI
authentication (service accounts, ADC, IAM) but different request/response formats
and endpoint paths.

The library already has `AnthropicClient` and OpenAI-compatible logic for formatting
requests correctly. The only thing that changes when running these models on Vertex AI
is the **transport layer**: the URL, auth headers, and endpoint action.

## Decision

Implement a `GoogleAdapter` class that wraps any existing `ProviderInterface` and
rewrites its HTTP transport to use Vertex AI infrastructure. Extract shared auth
logic from `GoogleClient` into a `GoogleAuthTrait` so both classes share it.

```
GoogleAuthTrait (ADC, JWT, service account, metadata server token exchange)
    ↑                         ↑
GoogleClient              GoogleAdapter
(Gemini format owned)     (delegates format to wrapped provider)
```

### API

```php
use WebFiori\Ai\Provider\Google\GoogleAdapter;
use WebFiori\Ai\Provider\Google\GoogleAdapterConfig;
use WebFiori\Ai\Provider\Anthropic\AnthropicClient;
use WebFiori\Ai\Provider\Anthropic\AnthropicClientConfig;

// Existing client owns format logic
$anthropic = new AnthropicClient(new AnthropicClientConfig(
    apiKey: '',
    model: 'claude-haiku-4-5@20251001',
));

// Adapter rewrites transport to Vertex AI
$client = new GoogleAdapter(
    provider: $anthropic,
    config: new GoogleAdapterConfig(
        projectId: 'my-project',
        location: 'us-east5',
        credentials: '/path/to/key.json',
    ),
);

// Transparent to callers
$response = $client->chat($messages, ['tools' => $tools]);
```

### How It Works

The `GoogleAdapter` intercepts HTTP at the `HttpClientInterface` level by wrapping
the inner provider's HTTP client with a `VertexHttpClientDecorator` that:

1. Rewrites the URL to the Vertex AI endpoint:
   ```
   {location}-aiplatform.googleapis.com/v1/projects/{project}/locations/{location}/publishers/{publisher}/models/{model}:{action}
   ```
2. Replaces the auth header with a Vertex IAM Bearer token (from `GoogleAuthTrait`)
3. Maps the action: `:rawPredict` for non-Google publishers

The inner provider (e.g., `AnthropicClient`) builds the request in its own format —
the adapter just redirects where it goes.

### GoogleAdapterConfig

```php
class GoogleAdapterConfig extends ClientConfig {
    public readonly string $projectId;
    public readonly string $location;
    public readonly string|array|null $credentials;
    public readonly ?string $accessToken;
    public readonly string $publisher; // 'anthropic', 'meta', 'mistralai'
}
```

### GoogleAuthTrait

Extracted from `GoogleClient`, provides:
- `getVertexAccessToken(): string`
- `exchangeJwtForAccessToken(string $jwt): string`
- `getTokenFromMetadataServer(): string`
- `resolveCredentials(): array`

### Composes with FallbackProvider

```php
$provider = new FallbackProvider([
    new GoogleAdapter($anthropicClient, $vertexConfig), // Claude on Vertex
    new AnthropicClient($directConfig),                  // Claude direct fallback
]);
```

## Alternatives Considered

**One client per publisher (`VertexAnthropicClient`, `VertexOpenAICompatClient`, etc.):**

```php
// Rejected approach
$client = new VertexAnthropicClient([
    'project_id' => 'my-project',
    'model' => 'claude-haiku-4-5@20251001',
    ...
]);
```

Rejected because:
- Duplicates Anthropic Messages API format logic already in `AnthropicClient`
- Bug fixes in Anthropic format must be applied to both `AnthropicClient` and `VertexAnthropicClient`
- 3-4 new client classes to maintain indefinitely
- Adding a new publisher requires a new class

**Single unified VertexAIClient handling all publishers:**
A single client that detects publisher from model name and branches format logic.
Rejected because the format logic for each publisher is substantial — it would become
a god class. The adapter pattern keeps each concern in its natural place.

## Consequences

**Easier:**
- New Model Garden publishers need zero new classes — just use `GoogleAdapter` with the appropriate inner provider
- Format bug fixes in `AnthropicClient` automatically apply to Vertex AI usage
- Composes with `FallbackProvider` for resilient multi-provider setups
- Auth logic is maintained in one place (`GoogleAuthTrait`)

**Harder:**
- HTTP interception requires a decorator on the inner provider's HTTP client
- The adapter must know the publisher name to build the correct Vertex URL
- Some providers may set auth headers that need to be overridden (e.g., Anthropic's `x-api-key`)
