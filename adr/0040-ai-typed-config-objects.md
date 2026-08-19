# ADR-0040: AI: Typed Configuration Objects for Provider Clients

**Date:** 2026-08-19
**Status:** Accepted

## Context

Provider clients currently accept configuration as associative arrays with string literal keys:

```php
$client = new GoogleClient([
    'api_key' => '...',
    'model' => 'gemini-2.5-flash',
    'project_id' => 'my-project',
]);
```

This approach has several problems:

1. **No discoverability** — Developers must read documentation to know valid keys
2. **No type safety** — Typos in keys silently fail (`'api_ky'` vs `'api_key'`)
3. **No IDE support** — No autocomplete, no refactoring, no "find usages"
4. **No validation** — Invalid values only fail at runtime deep in the call stack
5. **No documentation** — PHPDoc on arrays is limited

As we add more configuration options (e.g., `api_version` for Google Interactions API), the problem compounds.

## Decision

Replace associative array configuration with **typed configuration objects**.

### Base Configuration

```php
namespace WebFiori\Ai\Provider;

abstract class ClientConfig {
    public function __construct(
        public readonly string $model,
        public readonly ?int $timeout = 30,
        public readonly ?int $connectTimeout = 10,
    ) {}
}
```

### Provider-Specific Configuration

```php
namespace WebFiori\Ai\Provider\Google;

class GoogleClientConfig extends ClientConfig {
    public function __construct(
        string $model,
        public readonly ?string $apiKey = null,
        public readonly ?string $projectId = null,
        public readonly ?string $location = null,
        public readonly ?string $accessToken = null,
        public readonly GoogleApiVersion $apiVersion = GoogleApiVersion::AUTO,
        ?int $timeout = 30,
        ?int $connectTimeout = 10,
    ) {
        parent::__construct($model, $timeout, $connectTimeout);
    }
}

enum GoogleApiVersion: string {
    case AUTO = 'auto';
    case GENERATE_CONTENT = 'generate_content';
    case INTERACTIONS = 'interactions';
}
```

### Client Constructor

```php
class GoogleClient extends AbstractClient {
    public function __construct(GoogleClientConfig $config) {
        // ...
    }
}
```

### Usage

```php
$client = new GoogleClient(new GoogleClientConfig(
    model: 'gemini-2.5-flash',
    apiKey: getenv('GOOGLE_API_KEY'),
));

// With explicit API version
$client = new GoogleClient(new GoogleClientConfig(
    model: 'gemini-3.5-flash',
    apiKey: getenv('GOOGLE_API_KEY'),
    apiVersion: GoogleApiVersion::INTERACTIONS,
));
```

### All Provider Configs

| Provider | Config Class | Specific Fields |
|----------|--------------|-----------------|
| Base | `ClientConfig` | `model`, `timeout`, `connectTimeout` |
| Google | `GoogleClientConfig` | `apiKey`, `projectId`, `location`, `accessToken`, `apiVersion` |
| OpenAI | `OpenAIClientConfig` | `apiKey`, `organization`, `baseUrl` |
| Anthropic | `AnthropicClientConfig` | `apiKey`, `anthropicVersion` |
| Bedrock | `BedrockClientConfig` | `region`, `accessKeyId`, `secretAccessKey`, `sessionToken`, `profile` |

### Enums for Fixed-Option Values

Where configuration values have a fixed set of valid options, use enums:

```php
enum GoogleApiVersion: string {
    case AUTO = 'auto';
    case GENERATE_CONTENT = 'generate_content';
    case INTERACTIONS = 'interactions';
}

enum BedrockInvocationMethod: string {
    case CONVERSE = 'converse';
    case INVOKE = 'invoke';
    case RESPONSES = 'responses';
}
```

## Alternatives Considered

**Keep arrays, add constants for keys:**
```php
$client = new GoogleClient([
    GoogleConfig::API_KEY => '...',
    GoogleConfig::MODEL => '...',
]);
```
Rejected because it still lacks type safety on values and validation.

**Builder pattern:**
```php
$config = GoogleClientConfig::builder()
    ->model('gemini-2.5-flash')
    ->apiKey('...')
    ->build();
```
Rejected as unnecessarily verbose. PHP 8's named arguments provide the same flexibility with less code.

**Keep arrays for backward compatibility:**
Rejected because we are pre-v1.0 and should make breaking changes now rather than carry technical debt into the stable release.

## Consequences

**Easier:**
- IDE autocomplete shows all available options
- Type errors caught at compile time (static analysis)
- Refactoring is safe — rename a property and all usages update
- Self-documenting — config class is the documentation
- Enums prevent invalid values

**Harder:**
- Breaking change — all existing code must be updated
- Slightly more verbose (but named arguments help)
- Each provider needs its own config class

**Migration:**

Before:
```php
$client = new GoogleClient([
    'api_key' => '...',
    'model' => 'gemini-2.5-flash',
]);
```

After:
```php
$client = new GoogleClient(new GoogleClientConfig(
    apiKey: '...',
    model: 'gemini-2.5-flash',
));
```
