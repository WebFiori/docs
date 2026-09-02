# AI Configuration

Each provider client is constructed with a typed configuration object that extends `ClientConfig`. Common options (`model`, `timeout`, `connectTimeout`) are shared; the rest are provider-specific. This page is the configuration reference for all four providers.

<meta name="description" content="Configuration reference for WebFiori AI provider clients: OpenAI, Google (Gemini/Vertex), Anthropic, and AWS Bedrock, including authentication options.">

## Common Options

Every config class extends `WebFiori\Ai\Provider\ClientConfig` and shares these options:

| Option           | Type     | Default              | Description                          |
|------------------|----------|----------------------|--------------------------------------|
| `model`          | `string` | *(provider default)* | Default model for chat completions.  |
| `timeout`        | `int`    | `30`                 | Request timeout in seconds.          |
| `connectTimeout` | `int`    | `10`                 | Connection timeout in seconds.       |

## OpenAI — `OpenAIClientConfig`

```php
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;

$client = new OpenAIClient(new OpenAIClientConfig(
    apiKey: 'sk-...',
    model: 'gpt-4o',
));
```

| Option           | Type      | Default                     | Description                                  |
|------------------|-----------|-----------------------------|----------------------------------------------|
| `apiKey`         | `string`  | *(required)*                | OpenAI API key.                              |
| `model`          | `string`  | `gpt-4o`                    | Default chat model.                          |
| `organization`   | `?string` | `null`                      | OpenAI organization ID.                      |
| `baseUrl`        | `string`  | `https://api.openai.com/v1` | Override for Azure OpenAI / compatible APIs. |
| `embeddingModel` | `string`  | `text-embedding-3-small`    | Model used by `embed()`.                     |
| `imageModel`     | `string`  | `dall-e-3`                  | Model used by `generateImage()`.             |

The configurable `baseUrl` also lets you target any OpenAI-compatible server, including self-hosted models — see [Providers](learn/ai-providers).

## Google — `GoogleClientConfig`

Supports both the Gemini API (Google AI Studio) and Vertex AI. Authentication priority is `apiKey` > `accessToken` > `credentials`.

```php
use WebFiori\Ai\Provider\Google\GoogleClient;
use WebFiori\Ai\Provider\Google\GoogleClientConfig;

// Gemini API (API key)
$client = new GoogleClient(new GoogleClientConfig(
    apiKey: 'your-gemini-key',
    model: 'gemini-2.5-flash',
));
```

| Option           | Type                   | Default                                     | Description                                     |
|------------------|------------------------|---------------------------------------------|-------------------------------------------------|
| `model`          | `string`               | `gemini-2.5-flash`                          | Default chat model.                             |
| `apiKey`         | `?string`              | `null`                                      | Gemini API key from Google AI Studio.           |
| `projectId`      | `?string`              | `null`                                      | GCP project ID (required for Vertex AI).        |
| `location`       | `string`               | `global`                                    | GCP region, or `global` for automatic routing.  |
| `credentials`    | `string\|array\|null`  | `null`                                      | Service-account JSON path or array.             |
| `accessToken`    | `?string`              | `null`                                      | Pre-fetched OAuth2 access token.                |
| `api`            | `GoogleApi`            | `GoogleApi::GEMINI`                         | `GEMINI` or `VERTEX` endpoint.                  |
| `apiVersion`     | `GoogleApiVersion`     | `AUTO`                                       | `AUTO` detects the Interactions API for gemini-3.x. |
| `embeddingModel` | `string`               | `text-embedding-004`                        | Model used by `embed()`.                        |
| `imageModel`     | `string`               | `gemini-2.5-flash-preview-image-generation` | Model used by `generateImage()`.                |
| `publisher`      | `string`               | `google`                                    | Vertex Model Garden publisher (`anthropic`, `meta`, ...). |

## Anthropic — `AnthropicClientConfig`

```php
use WebFiori\Ai\Provider\Anthropic\AnthropicClient;
use WebFiori\Ai\Provider\Anthropic\AnthropicClientConfig;

$client = new AnthropicClient(new AnthropicClientConfig(
    apiKey: 'sk-ant-...',
    model: 'claude-sonnet-4-20250514',
));
```

| Option             | Type     | Default                     | Description                  |
|--------------------|----------|-----------------------------|------------------------------|
| `apiKey`           | `string` | *(required)*                | Anthropic API key.           |
| `model`            | `string` | `claude-sonnet-4-20250514`  | Default chat model.          |
| `maxTokens`        | `int`    | `4096`                      | Default max response tokens. |
| `baseUrl`          | `string` | `https://api.anthropic.com` | API base URL.                |
| `anthropicVersion` | `string` | `2023-06-01`                | API version header value.    |

## AWS Bedrock — `BedrockClientConfig`

Supports API-key auth or SigV4 (access/secret keys, session token, or a named AWS profile).

```php
use WebFiori\Ai\Provider\Bedrock\BedrockClient;
use WebFiori\Ai\Provider\Bedrock\BedrockClientConfig;
use WebFiori\Ai\Provider\Bedrock\ApiMethod;

$client = new BedrockClient(new BedrockClientConfig(
    region: 'us-east-1',
    accessKey: 'AKIA...',
    secretKey: 'wJal...',
    apiMethod: ApiMethod::CONVERSE,
));
```

| Option         | Type               | Default                                     | Description                                 |
|----------------|--------------------|---------------------------------------------|---------------------------------------------|
| `region`       | `string`           | *(required)*                                | AWS region (e.g., `us-east-1`).             |
| `model`        | `string`           | `anthropic.claude-3-5-sonnet-20241022-v2:0` | Default chat model.                         |
| `apiKey`       | `?string`          | `null`                                      | Bedrock API key (simple auth).             |
| `accessKey`    | `?string`          | `null`                                      | AWS access key ID (SigV4).                  |
| `secretKey`    | `?string`          | `null`                                      | AWS secret access key (SigV4).              |
| `sessionToken` | `?string`          | `null`                                      | AWS session token (temporary credentials).  |
| `profile`      | `?string`          | `null`                                      | AWS profile name for the credential chain.  |
| `apiMethod`    | `ApiMethod\|string`| `ApiMethod::CONVERSE`                        | Invocation API (`CONVERSE` or `INVOKE`).    |
| `maxTokens`    | `int`              | `4096`                                      | Default max response tokens.                |

> **Since 1.0.** `ApiMethod` is a string-backed enum. Pass a case such as `ApiMethod::CONVERSE` (recommended) or its string value `'converse'`.

## Keeping Secrets Out of Code

Read credentials from the environment rather than hard-coding them:

```php
$client = new OpenAIClient(new OpenAIClientConfig(
    apiKey: getenv('OPENAI_API_KEY') ?: '',
    model: 'gpt-4o',
));
```

## Where to Next

- [Basic Chat](learn/ai-basic-chat) — send messages and read responses
- [Providers](learn/ai-providers) — feature matrix and switching providers
