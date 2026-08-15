# ADR-0035: AI: Provider-Native Built-In Tools via Typed Enums

**Date:** 2026-08-16
**Status:** Accepted

## Context

Every major AI provider ships built-in/native tools that the model can invoke
without a PHP handler. These are passed differently in each provider's API:

| Provider | Built-in Tools | API Format |
|----------|---------------|------------|
| Google Gemini | `googleSearch`, `codeExecution`, `urlContext` | Separate object alongside `functionDeclarations` |
| Anthropic | `computer`, `bash`, `text_editor` | Different schema from custom tools |
| OpenAI | `web_search_preview`, `code_interpreter`, `file_search` | `type` field instead of `function` |

The library's `chat()` options accept `ToolInterface[]` for custom function
calling but have no way to pass native tools. The only workaround is
subclassing a provider and overriding `buildChatRequest()`,
`buildStreamChatRequest()`, and `buildIncrementalChatRequest()` — three
methods — for every provider.

Additionally, Google's own APIs are inconsistent: the Gemini API allows
`googleSearch` and `functionDeclarations` in the same request, while Vertex AI
rejects this combination.

## Decision

Introduce a `BuiltInToolInterface` marker and per-provider PHP 8.1 backed enums.
Native tools are passed via a `built_in_tools` key in `chat()` options:

```php
$response = $client->chat($messages, [
    'tools'          => [$myCustomTool],              // ToolInterface[] — unchanged
    'built_in_tools' => [GoogleBuiltInTool::GOOGLE_SEARCH], // new
]);
```

**`BuiltInToolInterface`** is a marker interface with a single method:

```php
interface BuiltInToolInterface {
    public function getValue(): string;
}
```

**Per-provider enums** implement it:

```php
enum GoogleBuiltInTool: string implements BuiltInToolInterface {
    case GOOGLE_SEARCH  = 'google_search';
    case CODE_EXECUTION = 'code_execution';
    case URL_CONTEXT    = 'url_context';

    public function getValue(): string { return $this->value; }
}
```

Each provider's `buildChatRequest()` reads `options['built_in_tools']`, merges
them into the correct API format, and throws `UnsupportedFeatureException` for
any built-in tool it does not recognise.

**Vertex AI conflict:** When `GoogleBuiltInTool::GOOGLE_SEARCH` is passed to a
Vertex AI endpoint alongside custom `tools`, an `UnsupportedFeatureException`
is thrown with a clear message explaining the limitation. Silent dropping is
rejected because it would discard custom tools the developer explicitly
registered without any indication.

## Alternatives Considered

**String-based (`'built_in_tools' => ['google_search']`):**

Simpler — no new classes. Rejected because:
- No IDE autocomplete — developers must know exact string identifiers
- Typos are only caught at runtime
- No way to constrain which strings are valid per provider at the type level

**Extending `ToolInterface`:**

Add a `isBuiltIn(): bool` method and a `NativeToolAdapter` class. Rejected
because built-in tools have no PHP handler (`execute()`) — forcing them to
implement `ToolInterface` would require a dummy `execute()` that throws, which
is misleading.

**Provider-specific `chat()` options (`'google_search' => true`):**

Rejected — breaks the provider-agnostic interface design (ADR-0028). Code
written for one provider would not transfer to another.

## Consequences

**Easier:**
- IDE autocomplete reveals available built-in tools per provider
- Type system prevents passing a `GoogleBuiltInTool` to `AnthropicClient`
  (wrong enum type)
- Vertex AI limitation is surfaced immediately with a clear exception rather
  than a silent API error from Google
- `AbstractClient` requires no changes — `$options` is already forwarded to
  `buildChatRequest()`

**Harder:**
- New built-in tools added by providers require a new enum case and a library
  release — developers cannot use a brand-new provider tool until the enum is
  updated (mitigated by the string fallback option in each enum if needed later)
