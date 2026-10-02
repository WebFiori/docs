# ADR-0047: AI: AgentProfile Inheritance via `extends` Key

**Date:** 2026-08-26
**Status:** Proposed

## Context

When building multi-domain AI agents, each domain needs its own `AgentProfile`
with a unique identity and domain-specific instructions. However, domains often
share a common set of rules — output formatting, chart specifications, follow-up
action format, file analysis guidelines, and general behavioral instructions.

Currently, the only options are:

1. Duplicate the shared rules in every profile JSON (maintenance nightmare)
2. Implement custom merge logic in the consuming application's code

Neither is ideal. A real-world application with 4+ agent domains sharing ~60% of
their prompt rules results in thousands of characters of duplication across
profile files. Changes to shared rules require editing every profile.

The `extends` pattern is well-established in: `tsconfig.json`, ESLint configs,
Docker Compose, Renovate config, and Composer itself.

## Decision

Add an `extends` key to the AgentProfile JSON schema. When loading a profile via
`fromFile()`, the library resolves the base profile recursively and merges fields
using a deterministic, field-specific strategy. An optional `inheritance_strategy`
key allows child profiles to override the default merge behavior per field.

### Default Merge Strategy

| Field | Type | Default Strategy | Rationale |
|-------|------|------------------|-----------|
| `identity` | string | **replace** | Identity is unique per agent |
| `output_format` | string\|null | **replace** | Format is domain-specific |
| `context` | string\|array\|null | **concat** | Background knowledge accumulates; strings normalized to arrays |
| `skills` | string[] | **concat** | Capabilities are additive — specialists still have general skills |
| `instructions` | string[] | **concat** | Rules accumulate: general → specific |
| `constraints` | string[] | **concat** | All constraints apply cumulatively |
| `examples` | array[] | **concat** | More examples = better few-shot |
| `tools` | string[] | **concat** | Tools are additive |
| `metadata` | assoc array | **merge** | Shallow `array_merge($base, $child)` |

Mental model: **arrays concat, scalars replace, metadata merges**.

### Per-Field Override: `inheritance_strategy`

Child profiles may declare an `inheritance_strategy` object to override the
default merge behavior for specific fields:

```json
{
    "extends": "support-base",
    "inheritance_strategy": {
        "skills": "replace",
        "instructions": "replace"
    },
    "identity": "Minimal agent with limited capabilities",
    "skills": ["Only basic troubleshooting"],
    "instructions": ["Only these instructions apply"]
}
```

**Allowed strategy values:**

| Value | Behavior | Applicable to |
|-------|----------|---------------|
| `concat` | base + child | Array fields |
| `replace` | Child wins, base discarded | All fields |
| `merge` | Shallow `array_merge` | `metadata` only |

**Rules:**
- `inheritance_strategy` applies only to the immediate merge (child with its
  direct parent). It is NOT inherited — each level declares its own.
- If `inheritance_strategy` is not present, defaults apply.
- Invalid field names or strategy values throw `RuntimeException`.
- `inheritance_strategy` is consumed during resolution and not present in the
  final `AgentProfile` object.

**Example — replace instructions in a deep chain:**

```
_base.json             → instructions: ["rule A", "rule B"]
support-base.json      → extends _base, instructions: ["rule C"]
                         Resolved: ["rule A", "rule B", "rule C"]
tier1.json             → extends support-base
                         inheritance_strategy: {"instructions": "replace"}
                         instructions: ["only rule D"]
                         Resolved: ["only rule D"]
```

### Resolution Rules

1. `extends` value is a filename stem (no `.json` extension), resolved relative
   to the child profile's directory
2. Chained inheritance is supported — resolved recursively
3. Circular references throw `RuntimeException` with the full chain path
4. Missing base file throws `RuntimeException` with the resolved path

```php
// Resolution example:
// /agents/tier1-support.json has "extends": "support-base"
// Resolves to: /agents/support-base.json
//
// /agents/support-base.json has "extends": "_base"
// Resolves to: /agents/_base.json
//
// /agents/_base.json has no "extends"
// Chain complete: _base → support-base → tier1-support
```

### API

**`fromFile()` — automatic resolution:**

```php
use WebFiori\Ai\Tool\AgentProfile;

// Automatically resolves "extends" chain
$profile = AgentProfile::fromFile('/agents/tier1-support.json');
```

**`fromArray()` — optional `$basePath` parameter:**

```php
// $basePath tells fromArray() where to resolve "extends" from
$profile = AgentProfile::fromArray($data, basePath: '/agents/');

// Without $basePath, "extends" is ignored (backward compatible)
$profile = AgentProfile::fromArray($data);
```

**`merge()` — public utility for programmatic merging:**

```php
$base = AgentProfile::fromFile('/agents/_base.json');
$child = AgentProfile::fromFile('/agents/support.json');

// Manual merge using default strategies
$merged = AgentProfile::merge(base: $base, child: $child);

// Manual merge with strategy overrides
$merged = AgentProfile::merge(
    base: $base,
    child: $child,
    strategies: ['skills' => 'replace', 'instructions' => 'replace'],
);
```

### Full Example

**`/agents/_base.json`:**

```json
{
    "instructions": ["Use markdown formatting", "Be concise"],
    "constraints": ["Never reveal internal system details"],
    "output_format": "Respond in markdown.",
    "skills": ["Answer questions clearly", "Follow company tone"],
    "metadata": {"org": "WebFiori", "version": "1.0"}
}
```

**`/agents/support-base.json`:**

```json
{
    "extends": "_base",
    "identity": "You are a customer support agent.",
    "skills": ["Search knowledge base", "Check order status", "Escalate tickets"],
    "instructions": ["Always greet the customer by name", "Check order history first"],
    "constraints": ["Never promise refunds without approval"],
    "tools": ["search_orders", "search_kb"]
}
```

**`/agents/tier1-support.json`:**

```json
{
    "extends": "support-base",
    "instructions": ["Escalate to tier-2 after 3 failed resolution attempts"],
    "constraints": ["Cannot issue refunds over $50"],
    "metadata": {"tier": "1", "version": "1.1"}
}
```

**Resolved result for `tier1-support`:**

```
identity:      "You are a customer support agent."
skills:        ["Answer questions clearly", "Follow company tone",
                "Search knowledge base", "Check order status", "Escalate tickets"]
instructions:  ["Use markdown formatting", "Be concise",
                "Always greet the customer by name", "Check order history first",
                "Escalate to tier-2 after 3 failed resolution attempts"]
constraints:   ["Never reveal internal system details",
                "Never promise refunds without approval",
                "Cannot issue refunds over $50"]
output_format: "Respond in markdown."
tools:         ["search_orders", "search_kb"]
metadata:      {"org": "WebFiori", "version": "1.1", "tier": "1"}
examples:      []
```

**Example with `inheritance_strategy` override:**

**`/agents/minimal-bot.json`:**

```json
{
    "extends": "support-base",
    "inheritance_strategy": {
        "skills": "replace",
        "instructions": "replace",
        "constraints": "replace"
    },
    "identity": "Minimal FAQ bot.",
    "skills": ["Answer FAQs"],
    "instructions": ["Only answer from the FAQ list"],
    "constraints": ["Cannot escalate", "Cannot access order data"]
}
```

**Resolved result for `minimal-bot`:**

```
identity:      "Minimal FAQ bot."
skills:        ["Answer FAQs"]
instructions:  ["Only answer from the FAQ list"]
constraints:   ["Cannot escalate", "Cannot access order data"]
output_format: "Respond in markdown."
tools:         ["search_orders", "search_kb"]
metadata:      {"org": "WebFiori", "version": "1.0"}
examples:      []
```

Note: `tools` and `metadata` were not overridden in `inheritance_strategy`, so
they still use default behavior (concat and merge respectively).

### Implementation

Core resolution is a private recursive method in `AgentProfile`:

```php
private static function resolveInheritance(string $path, array $visited = []): array {
    $realPath = realpath($path);

    if ($realPath === false) {
        throw new RuntimeException('Profile file not found: ' . $path);
    }

    if (in_array($realPath, $visited, true)) {
        $chain = implode(' → ', array_map('basename', $visited));
        throw new RuntimeException(
            'Circular profile inheritance detected: ' . $chain . ' → ' . basename($realPath)
        );
    }

    $visited[] = $realPath;
    $contents = file_get_contents($realPath);
    $data = json_decode($contents, true);

    if (!is_array($data)) {
        throw new RuntimeException('Invalid JSON in profile file: ' . $path);
    }

    if (!isset($data['extends']) || $data['extends'] === '') {
        unset($data['extends']);
        return $data;
    }

    $baseFile = dirname($realPath) . '/' . $data['extends'] . '.json';
    $baseData = self::resolveInheritance($baseFile, $visited);

    $strategies = $data['inheritance_strategy'] ?? [];
    unset($data['extends'], $data['inheritance_strategy']);

    return self::mergeProfileData($baseData, $data, $strategies);
}

private static function mergeProfileData(array $base, array $child, array $strategies = []): array {
    $defaults = [
        'identity'      => 'replace',
        'output_format' => 'replace',
        'context'       => 'concat',
        'skills'        => 'concat',
        'instructions'  => 'concat',
        'constraints'   => 'concat',
        'examples'      => 'concat',
        'tools'         => 'concat',
        'metadata'      => 'merge',
    ];

    $effective = array_merge($defaults, $strategies);
    $result = [];

    foreach ($defaults as $field => $defaultStrategy) {
        $strategy = $effective[$field];

        switch ($strategy) {
            case 'replace':
                if ($field === 'identity') {
                    $result[$field] = ($child[$field] ?? '') !== ''
                        ? $child[$field]
                        : ($base[$field] ?? '');
                } else {
                    $result[$field] = array_key_exists($field, $child)
                        ? $child[$field]
                        : ($base[$field] ?? null);
                }
                break;

            case 'concat':
                $result[$field] = array_merge(
                    $base[$field] ?? [],
                    $child[$field] ?? []
                );
                break;

            case 'merge':
                $result[$field] = array_merge(
                    $base[$field] ?? [],
                    $child[$field] ?? []
                );
                break;

            default:
                throw new RuntimeException(
                    "Invalid inheritance strategy '{$strategy}' for field '{$field}'"
                );
        }
    }

    return $result;
}
```

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| No `extends` key | No-op, loads normally (backward compatible) |
| `extends` is empty string | Treated as no inheritance |
| Base file missing | `RuntimeException` with resolved path |
| Circular: A → B → A | `RuntimeException` with chain description |
| Deep chain: A → B → C → D | Resolved recursively, all merges applied |
| `fromArray()` without `$basePath` | `extends` key is ignored |
| `fromUrl()` with `extends` | Not supported — no filesystem context |
| Child has empty `identity` (`""`) | Base identity is used |
| `inheritance_strategy` with invalid field | `RuntimeException` |
| `inheritance_strategy` with invalid value | `RuntimeException` |
| `inheritance_strategy` is not inherited | Each level declares its own |

## Alternatives Considered

**Application-level merge:**
Works but forces every consumer to reimplement the same logic. Not DRY. The
merge strategy is non-trivial (field-specific rules) and error-prone to
implement repeatedly.

**PHP class inheritance:**
Doesn't apply since profiles are JSON-driven, not class-driven. Users define
profiles as data files, not PHP classes.

**Template variables in JSON:**
Using `{{base.instructions}}` placeholders. More complex to implement, less
intuitive, requires a template engine, and doesn't compose well for arrays
(concat vs replace depends on the field).

**Fixed strategy with no override:**
Simpler but too rigid. A child that needs to start fresh on instructions
(e.g., a minimal bot) would have no escape hatch. Per-field override solves
this without adding complexity to simple profiles.

**`inheritance_strategy` is inherited down the chain:**
Considered making strategy declarations propagate to descendants. Rejected —
each level should independently control how it merges with its parent. Inheriting
strategies creates confusing implicit behavior in deep chains.

**`$include` with multiple bases:**
Multiple inheritance from several files. Rejected — significantly more complex
(diamond problem, merge order ambiguity) with limited practical benefit. Single
inheritance with chaining covers real use cases.

## Consequences

**Easier:**
- Multi-domain agent applications share rules without duplication
- Changes to shared rules propagate automatically to all child profiles
- Profiles stay small and focused on domain-specific differences
- Familiar pattern for developers (tsconfig, ESLint, Docker Compose)
- `inheritance_strategy` provides escape hatch without sacrificing simple defaults
- Fully backward compatible — existing profiles work unchanged

**Harder:**
- Debugging the final merged profile requires understanding the chain
- `fromUrl()` cannot use inheritance (no filesystem context)
- `fromArray()` needs the caller to provide `$basePath` for resolution
- Deep inheritance chains (4+ levels) may be confusing to maintain
- No way to remove a *specific* item from a base array (only replace the entire field)
