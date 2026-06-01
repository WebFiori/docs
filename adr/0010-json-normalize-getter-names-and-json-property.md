# ADR-0010: Normalize Getter-Derived Property Names and Add #[JsonProperty] Attribute

**Date:** 2026-06-02
**Status:** Proposed

## Context

When auto-mapping objects, getter-derived names and public property names use different conventions, producing inconsistent casing in the same JSON output:

- **Getters**: strip `get` prefix, keep remainder as-is → `getName()` becomes `"Name"` (PascalCase)
- **Public properties**: use exact field name → `public string $sku` becomes `"sku"`

```php
class Product {
    public string $sku = 'ABC-001';

    public function getName(): string { return 'Keyboard'; }
    public function getUnitPrice(): float { return 49.99; }
}

$json = new Json();
$json->addObject('product', new Product());
echo $json;
// {"product":{"Name":"Keyboard","UnitPrice":49.99,"sku":"ABC-001"}}
// Mixed PascalCase and lowercase in the same object
```

The root cause: `substr($methods[$y], 3)` produces PascalCase names that are not passed through `CaseConverter` before being added.

## Decision

Two changes:

### 1. Normalize getter-derived names through CaseConverter

When stripping the `get` prefix, treat the remainder as a PascalCase word and pass it through `CaseConverter::convert()` using the active props style:

- Style `camel`: `getName()` → `name`, `getUnitPrice()` → `unitPrice`
- Style `snake`: `getName()` → `name`, `getUnitPrice()` → `unit_price`
- Style `kebab`: `getName()` → `name`, `getUnitPrice()` → `unit-price`
- Style `none`: keep as-is (`Name`, `UnitPrice`) — preserves backward compatibility

After the change with style `camel`:

```php
$json = new Json([], 'camel');
$json->addObject('product', new Product());
echo $json;
// {"product":{"name":"Keyboard","unitPrice":49.99,"sku":"ABC-001"}}
```

### 2. Add `#[JsonProperty]` attribute for explicit name override

Applicable to both getters and public properties. Overrides automatic name derivation and style conversion:

```php
namespace WebFiori\Json;

#[\Attribute(\Attribute::TARGET_METHOD | \Attribute::TARGET_PROPERTY)]
class JsonProperty {
    public function __construct(public readonly string $name) {}
}
```

Usage:

```php
use WebFiori\Json\JsonProperty;

class Product {
    #[JsonProperty('product_sku')]
    public string $sku = 'ABC-001';

    #[JsonProperty('price')]
    public function getUnitPrice(): float { return 49.99; }
}
// Output: {"product_sku":"ABC-001","price":49.99}
```

Implementation:

```php
// For getters:
$refMethod = new \ReflectionMethod($obj, $methods[$y]);
$attrs = $refMethod->getAttributes(JsonProperty::class);
if (!empty($attrs)) {
    $propName = $attrs[0]->newInstance()->name;
} else {
    $propName = substr($methods[$y], 3); // then normalize via CaseConverter
}

// For public properties:
$attrs = $prop->getAttributes(JsonProperty::class);
if (!empty($attrs)) {
    $propName = $attrs[0]->newInstance()->name;
} else {
    $propName = $prop->getName();
}
```

## Alternatives Considered

- **Always `lcfirst()` getter names** — simple but doesn't integrate with the style system for multi-word names like `UnitPrice`.
- **Only provide `#[JsonProperty]`** — forces manual annotation on every getter, bad DX for the common case.
- **Use docblock annotations** — PHP 8.1 native attributes are cleaner and have IDE/static analysis support.

## Consequences

- **Breaking change for styles other than `none`**: getter-derived names will be properly normalized (e.g. `Name` → `name` in camel style). With style `none`, behavior is unchanged.
- **Consistent output**: all properties in the same object follow the same naming convention.
- **Explicit override**: `#[JsonProperty]` gives full control when automatic derivation doesn't fit.
- **Matches ecosystem patterns**: Jackson's `@JsonProperty`, Symfony's `#[SerializedName]`, .NET's `[JsonPropertyName]`.

## Related

- [ADR-0009](0009-json-include-null-false-and-json-ignore.md) — `#[JsonIgnore]` attribute (same attribute system)
- [ADR-0011](0011-json-typed-deserialization.md) — `#[JsonType]` attribute (same attribute system)

## GitHub Issue

[webfiori/json#58](https://github.com/WebFiori/json/issues/58)
