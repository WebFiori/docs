# ADR-0011: Typed Deserialization With Nested Object Hydration

**Date:** 2026-06-02
**Status:** Accepted

## Context

The library can serialize objects to JSON but cannot deserialize JSON back to typed objects. `Json::decode()` always returns a `Json` instance. There is no way to hydrate domain classes from JSON data, preventing round-trip serialization.

```php
$decoded = Json::decode('{"username":"ibrahim","email":"ibrahim@example.com"}');
// Returns Json instance, NOT a User object
// No way to get a User object back
```

Enterprise serialization libraries (Jackson, System.Text.Json, Symfony Serializer) all support `deserialize($json, TargetClass)` to produce fully typed objects, including nested types resolved recursively.

## Decision

Three complementary mechanisms for typed deserialization:

### 1. `Json::decodeAs()` — class-level deserialization

```php
$user = Json::decodeAs('{"username":"ibrahim","email":"ibrahim@example.com"}', User::class);
// Returns User instance
```

### 2. `#[JsonType]` attribute — nested type mapping

For cases where PHP's type system is insufficient (arrays of objects, union types):

```php
namespace WebFiori\Json;

#[\Attribute(\Attribute::TARGET_PROPERTY | \Attribute::TARGET_PARAMETER)]
class JsonType {
    public function __construct(
        public readonly string $className,
        public readonly bool $isArray = false
    ) {}
}
```

Usage:

```php
class Order {
    public function __construct(
        private User $customer,                          // auto-resolved via reflection
        private Address $shippingAddress,                // auto-resolved via reflection
        #[JsonType(LineItem::class, isArray: true)]
        private array $items                             // needs annotation
    ) {}
}

$order = Json::decodeAs($jsonString, Order::class);
$order->getCustomer();          // User instance
$order->getShippingAddress();   // Address instance
$order->getItems();             // LineItem[] array
```

### 3. Type registry on Json instance — runtime mapping

For dynamic/runtime mapping without attributes:

```php
$json = Json::decode($jsonString);
$json->setTypeMap([
    'customer' => User::class,
    'shippingAddress' => Address::class,
    'items' => [LineItem::class],  // array notation = array of that type
]);
$json->get('customer');  // Returns User instance
$json->get('items');     // Returns LineItem[] array
```

### Nested type resolution priority

```
1. #[JsonType] attribute on property/parameter → explicit, always wins
2. Constructor parameter type hint via reflection → automatic for class types
3. Type registry → runtime override
4. No type info → stays as Json instance (backward compatible)
```

### Hydration strategy

```
decodeAs(jsonData, TargetClass):
  1. If TargetClass has static fromJSON(Json $json): self → call it (custom deserialization)
  2. Otherwise, reflect constructor parameters:
     a. Match JSON keys to parameter names
     b. For each parameter, determine type (attribute > reflection > registry)
     c. If type is a class → recursively decodeAs(value, Type)
     d. If type is array with #[JsonType] → recursively hydrate each element
     e. Otherwise → use scalar value directly
  3. Instantiate TargetClass with resolved parameters
  4. For remaining JSON keys not in constructor:
     a. Try setter methods: setUsername('value')
     b. Try public property assignment: $obj->username = 'value'
     c. Apply same recursive type resolution for setters/properties
```

### What auto-resolves vs what needs annotation

| Case | Auto-resolves? | Needs `#[JsonType]`? |
|------|---------------|----------------------|
| `User $customer` | ✅ via reflection | No |
| `?User $customer` | ✅ nullable | No |
| `array $items` (of objects) | ❌ | Yes |
| `mixed $data` | ❌ | Yes, if typed desired |
| Union types `User\|Admin` | ❌ | Yes |
| Deep nesting `Order → User → Role` | ✅ recursive | Only if each level has type info |

## Alternatives Considered

- **Only support `JsonI` for custom deserialization** — requires implementing an interface on every class, too invasive for third-party objects.
- **Only attribute-based** — forces annotation on every property even when reflection suffices.
- **Only runtime registry** — no compile-time safety, easy to forget mappings.
- **Docblock `@var` parsing** — fragile, no IDE enforcement, superseded by native types and attributes.

## Consequences

- **Non-breaking**: `Json::decode()` continues to return `Json` instances. `Json::decodeAs()` is a new method. `get()` only returns typed objects when a type registry is configured.
- **Round-trip enabled**: objects can be serialized and deserialized symmetrically.
- **`fromJSON()` mirrors `toJSON()`**: the `JsonI` interface gains a deserialization counterpart.
- **Reflection cost**: constructor and attribute inspection adds overhead. Should be mitigated with metadata caching for repeated deserialization of the same class.
- **Incomplete type info fails gracefully**: properties without type information remain as `Json` instances rather than throwing.

## Related

- [ADR-0009](0009-json-include-null-false-and-json-ignore.md) — `#[JsonIgnore]` attribute
- [ADR-0010](0010-json-normalize-getter-names-and-json-property.md) — `#[JsonProperty]` attribute

## GitHub Issue

[webfiori/json#59](https://github.com/WebFiori/json/issues/59)
