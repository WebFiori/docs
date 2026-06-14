# ADR-0009: Include All Getter Return Values and Add #[JsonIgnore] Attribute

**Date:** 2026-06-02
**Status:** Proposed

## Context

In `JsonConverter::objectToJson()`, getters returning `null` or `false` are silently excluded from serialization:

```php
if ($propVal !== false && $propVal !== null) {
    $json->add(substr($methods[$y], 3), $propVal);
}
```

This causes two problems:

1. **Boolean data loss** — `isActive()` returning `false` is valid data, not a skip signal.
2. **Inconsistency** — public properties with `null` values ARE included, but getters returning `null` are NOT.

```php
class User {
    public ?string $nickname = null;     // ✅ Included as null

    public function getMiddleName(): ?string {
        return null;                      // ❌ Silently excluded
    }

    public function isActive(): bool {
        return false;                     // ❌ Silently excluded
    }
}
```

There is no mechanism to explicitly exclude a property — the library overloads `null`/`false` as sentinel values for "don't serialize."

## Decision

Two changes:

### 1. Include all getter return values by default

Remove the `null`/`false` filter:

```php
$propVal = call_user_func([$obj, $methods[$y]]);
$json->add(substr($methods[$y], 3), $propVal);
```

### 2. Add `#[JsonIgnore]` attribute for explicit exclusion

Applicable to both getters and public properties:

```php
namespace WebFiori\Json;

#[\Attribute(\Attribute::TARGET_METHOD | \Attribute::TARGET_PROPERTY)]
class JsonIgnore {}
```

Usage:

```php
use WebFiori\Json\JsonIgnore;

class User {
    #[JsonIgnore]
    public string $internalId = 'abc-123';  // excluded

    #[JsonIgnore]
    public function getDebugInfo(): string { // excluded
        return 'internal';
    }

    public function getName(): string {      // included
        return 'Ibrahim';
    }

    public ?string $email = null;            // included as null
}
```

Implementation checks for the attribute via reflection:

```php
// For getters:
$refMethod = new \ReflectionMethod($obj, $methods[$y]);
if (!empty($refMethod->getAttributes(JsonIgnore::class))) {
    continue;
}

// For public properties:
if (!empty($prop->getAttributes(JsonIgnore::class))) {
    continue;
}
```

## Alternatives Considered

- **Only skip null, include false** — fixes booleans but still can't serialize null from getters.
- **Configurable null strategy (`nullStrategy: 'include'`)** — adds complexity without solving the opt-out problem.
- **Return a sentinel object to signal skip** — non-standard, awkward API.

## Consequences

- **Breaking change**: objects that relied on returning `null`/`false` to hide properties will now expose them. Developers must migrate to `#[JsonIgnore]` for explicit exclusion.
- **Consistency**: getters and public properties now follow the same inclusion rules.
- **Explicit over implicit**: exclusion is declared, not inferred from return values.
- **PHP 8.1 required**: already the minimum version, so native attributes are available.
- **Foundation for attribute system**: pairs with `#[JsonProperty]` ([ADR-0010](0010-json-normalize-getter-names-and-json-property.md)) and `#[JsonType]` ([ADR-0011](0011-json-typed-deserialization.md)).

## GitHub Issue

[webfiori/json#57](https://github.com/WebFiori/json/issues/57)
