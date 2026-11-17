# ADR-0062: AI: Cached Input Token Usage and Pricing

**Date:** 2026-11-17
**Status:** Accepted

## Context

All supported providers can serve part of a prompt from a cache and bill those
tokens at a lower rate (Gemini implicit caching is on by default and discounts
cached input by 90%). Every provider reports how many input tokens were cached,
but the library drops that information:

- `Usage` holds only `promptTokens` and `completionTokens`.
- No provider reads its cached-token fields (`cachedContentTokenCount`,
  `total_cached_tokens`, `prompt_tokens_details.cached_tokens`,
  `cache_read_input_tokens`, `cacheReadInputTokens`).
- `PricingConfig` has one input price per model, so
  `AbstractClient::calculateCost()` charges cached tokens at the full input
  price and overstates cost whenever caching happens.
- Audit entries (ADR-0038) and metrics events (ADR-0036) report only the full
  prompt total, so cost allocation built on them is overstated too.

Providers also disagree on what "input tokens" means:

| Provider | Reported input field | Includes cached tokens? |
|---|---|---|
| Google (`generateContent`, Interactions) | `promptTokenCount`, `total_input_tokens` | Yes |
| OpenAI (Chat Completions) | `prompt_tokens` | Yes |
| Anthropic | `input_tokens` | **No**, cache reads and writes are separate counters |
| Bedrock Converse | `inputTokens` | **No**, cache reads and writes are separate counters |

As a result `getPromptTokens()` undercounts the real input for Anthropic and
Bedrock whenever caching occurs, and the same number means different things
depending on the provider.

Tracked by epic #196 (sub-issues #197 to #201).

## Decision

### 1. Uniform token semantics

`Usage::getPromptTokens()` is the total input sent, including cache reads and
cache writes, for every provider. Two new counts are subsets of it:

- `getCachedTokens()`: input tokens read from cache.
- `getCacheWriteTokens()`: input tokens written to cache by this request
  (0 when the provider does not report writes).
- Uncached input = prompt − cached − cache writes.

Providers that report only the non-cached part add the cache counters back:

```
Anthropic: prompt = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
Bedrock:   prompt = inputTokens  + cacheReadInputTokens    + cacheWriteInputTokens
```

This is the only behavior change for existing users. It corrects an undercount
and differs from today's numbers only when caching occurs.

### 2. `Usage` clamps instead of throwing

The constructor takes `int $cachedTokens = 0, int $cacheWriteTokens = 0`.
If `cached + writes > prompt`, cached tokens are reduced first, then writes, so
the subset invariant always holds and `promptTokens` is never altered.
Throwing was rejected: bad provider data should not fail a request that already
succeeded and was billed.

### 3. Fixed `toArray()` shape

`toArray()` always emits `cached_tokens` and `cache_write_tokens`, including
when 0. `fromArray()` defaults missing keys to 0, so payloads written before
this change (recorded responses per ADR-0042, cached responses per ADR-0033 and
ADR-0049) still load.

### 4. `PricingConfig`: optional cached prices

`setModel()` gains optional trailing parameters, and the constructor array
accepts optional keys:

```php
$pricing = new PricingConfig([
    'claude-sonnet-4' => [
        'input' => 3.00, 'output' => 15.00,
        'cached_input' => 0.30, 'cache_write' => 3.75,
    ],
]);

$pricing->setModel('gpt-4o', 2.50, 10.00, cachedInputPrice: 1.25);
```

New getters `getCachedInputPrice(string $model): ?float` and
`getCacheWritePrice(string $model): ?float` return `null` when unset. A missing
price falls back to the normal input price, so existing configurations produce
the same cost as before. `hasModel()` still requires only `input` and `output`.
There are no built-in default prices, consistent with ADR-0034's rule that
model-dependent numbers are configured by the user.

### 5. Cost calculation and `CostResult`

```
uncached  = prompt − cached − writes
inputCost = uncached × input
          + cached   × (cachedInput ?? input)
          + writes   × (cacheWrite  ?? input)
```

`CostResult` gains optional constructor arguments `cachedInputCost` and
`cacheWriteCost` (default 0.0) with getters `getCachedInputCost()` and
`getCacheWriteCost()`. `getInputCost()` remains the total input cost and
`getTotal()` is unchanged, so existing callers keep their meaning.

### 6. Audit and metrics

The success audit entry of `chat()` adds `cached` and `cache_write` to its
`tokens` array. The `request.completed` metrics event and the completion log
entry add `cached_tokens` and `cache_write_tokens`. `prompt` keeps its meaning
(total input). Failure and cancellation paths keep their null values.

`streamChat()` audit entries are unchanged: they record only an estimated
completion count and carry no provider usage, so there is nothing cached to
report. `embed()` is unchanged as well.

### 7. Provider mapping

Each provider maps its fields onto the semantics above, for streaming and
non-streaming responses:

- Google: `usageMetadata.cachedContentTokenCount` and
  `usage.total_cached_tokens` (Interactions). Prompt totals already include
  cached tokens.
- OpenAI: `usage.prompt_tokens_details.cached_tokens`. Streaming requests now
  send `stream_options: {"include_usage": true}`. Without it, Chat Completions
  streams carry no usage, so streamed responses reported none at all. The final
  usage chunk has an empty `choices` array and is read before the
  empty-choices early return.
- Anthropic: `cache_read_input_tokens`, `cache_creation_input_tokens`; streaming
  input usage arrives in `message_start`.
- Bedrock: Converse `cacheReadInputTokens` and `cacheWriteInputTokens`;
  InvokeModel uses the Anthropic mapping for Anthropic models.

## Alternatives Considered

**One large issue for all providers.** Harder to review and ship incrementally.
Rejected in favor of a core change first, then one change per provider.

**Expose raw provider usage arrays only.** Gives visibility, but every caller
would need provider-specific code to compute cost or hit rates.

**Keep `getPromptTokens()` as each provider reports it.** Leaves totals
inconsistent across providers, which makes cost and context accounting wrong for
Anthropic and Bedrock (and for `ContextUsage`, ADR-0048, which uses exact
`promptTokens` from a response).

**A separate `CacheUsage` object on `ChatResponse`.** Leaves `Usage` unchanged
but splits related numbers across two objects and complicates cost calculation.

**A cached-price multiplier instead of an absolute price.** Providers publish
absolute prices and the ratio varies by model and cache lifetime, so absolute
prices are simpler to configure correctly.

**Throw when cached + writes exceed prompt.** See decision 2.

**Emit the new `toArray()` keys only when non-zero.** The array shape would
depend on the data, forcing consumers to check key existence.

**Replace `prompt` in audit entries with the uncached count.** Silently changes
what existing consumers read.

**Introduce a `ModelPricing` value object now.** See below.

## Consequences

**Easier:**
- Callers see cache hit rates through `Usage`, and a cost that reflects cache
  discounts, with the breakdown available on `CostResult`.
- `getPromptTokens()` means the same thing on every provider.
- Audit and metrics consumers can do accurate cost allocation.
- Everything is additive. Old serialized payloads load, and pricing tables
  without cached prices produce the same costs.

**Harder:**
- Anthropic and Bedrock prompt-token totals increase when caching occurs. This
  is a correction but should be called out in the changelog.
- Tests asserting the exact `Usage::toArray()` shape need the two new keys.
- Anthropic cache-write pricing differs by TTL (5 minutes or 1 hour). A single
  `cache_write` price cannot express both. This limitation is documented and
  TTL-specific pricing is out of scope.
- Cached and cache-write prices are user-configured and can go stale, the same
  tradeoff as the rest of `PricingConfig`.
- OpenAI streaming requests gain a `stream_options` field. OpenAI and Gemini's
  OpenAI-compatible endpoint accept it; a third-party OpenAI-compatible server
  that rejects unknown fields would need it removed.
- Anthropic and Bedrock only cache when requests carry cache markers
  (`cache_control`, `cachePoint`), which the library does not send. They report
  0 cached tokens until cache-marker support is added. The mapping itself was
  verified against both live APIs using hand-built requests with markers.

### Future direction: `ModelPricing`

The nested-array pricing format, extended here with two more optional keys, is a
stopgap. The proper design is an immutable `ModelPricing` value object per model
(`input`, `output`, `cachedInput`, `cacheWrite`), owning the "unset falls back to
the input price" rule and price validation, with `PricingConfig` storing
`array<string, ModelPricing>`. New price dimensions, such as TTL-specific cache
writes, would then be new fields on one class instead of more `setModel()`
parameters and array keys.

It was not chosen for this change because the epic needs only two additional
optional values, and `PricingConfig::getModels()`, the array constructor and
`setModel()` are public API that would need dual-form support. The decisions
above stay compatible with that move: the array keys, the `setModel()`
parameters and the getters can all be kept and backed by `ModelPricing`
internally. A later ADR should introduce it once another price dimension is
needed.
