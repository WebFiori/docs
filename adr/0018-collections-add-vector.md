# ADR-0018: Add Vector to webfiori/collections

**Date:** 2026-06-03
**Status:** Proposed

## Context

The `webfiori/collections` package provides `LinkedList`, `Queue`, and `Stack`. The `LinkedList` is used as the child node storage in `webfiori/ui`'s `HTMLNode` class, where the dominant access pattern is index-based reads in loops:

```php
for ($x = 0; $x < $chCount; $x++) {
    $child = $children->get($x); // O(n) per call — traverses from head
}
```

This results in O(n²) performance for any operation that iterates children by index (rendering, searching by ID/tag, building parse trees). A native array provides O(1) index access, but exposing raw arrays loses the encapsulated API that the framework relies on (`get()`, `size()`, `indexOf()`, `remove()`).

The `php-ds` extension provides `Ds\Vector` with ideal characteristics, but requiring a PECL extension is not viable for a library.

## Decision

Add a `Vector` class to `webfiori/collections` that wraps a native PHP array and provides O(1) index-based access with an object-oriented API.

```php
namespace WebFiori\Collections;

class Vector implements \Countable, \Iterator {
    private array $elements = [];

    public function add(mixed $element): void;
    public function get(int $index): mixed;
    public function set(int $index, mixed $element): void;
    public function insert(mixed $element, int $index): void;
    public function removeAt(int $index): mixed;
    public function remove(mixed $element): mixed;
    public function indexOf(mixed $element): int;
    public function size(): int;
    public function clear(): void;
    public function replace(mixed $old, mixed $new): bool;
    public function toArray(): array;
    // Iterator interface methods
}
```

### Performance Characteristics

| Operation | Vector (array-backed) | LinkedList |
|-----------|----------------------|------------|
| `get($index)` | O(1) | O(n) |
| `add()` (append) | O(1) amortized | O(1) |
| `removeAt($index)` | O(n) splice | O(n) traverse |
| `indexOf()` | O(n) | O(n) |
| `insert()` at position | O(n) splice | O(n) traverse |
| `size()` | O(1) | O(1) |
| Iteration | O(n), cache-friendly | O(n), pointer chasing |

### Method Naming

Methods align with existing `LinkedList` conventions where possible (`add`, `get`, `size`, `indexOf`, `clear`, `replace`) to ease migration. New names (`set`, `removeAt`, `toArray`) fill gaps.

## Implementation Steps

| Step | Description | Issue |
|------|-------------|-------|
| 1 | Implement `Vector` class with full test coverage | [WebFiori/collections#4](https://github.com/WebFiori/collections/issues/4) |
| 2 | Replace `LinkedList` with `Vector` for HTMLNode child storage | [WebFiori/ui#66](https://github.com/WebFiori/ui/issues/66) |

## Alternatives Considered

1. **Use raw arrays directly in HTMLNode**: Loses encapsulation; `children()` would return an array that callers can mutate freely, breaking parent-child invariants.

2. **Require `php-ds` extension**: `Ds\Vector` is ideal but PECL extensions are not viable for a library that must install with `composer require` alone.

3. **Keep LinkedList, add index caching**: Complex, error-prone (cache invalidation on insert/remove), and still slower than native array for sequential access.

4. **Use `SplFixedArray`**: O(1) access but fixed size — requires resize on every add/remove, which is awkward for a dynamic child list.

## Consequences

### Benefits

- O(1) index access eliminates O(n²) patterns in `webfiori/ui` rendering and searching
- Reusable across all WebFiori packages that need ordered collections
- No external dependency — pure PHP, array-backed
- Method API familiar to existing `LinkedList` users

### Trade-offs

- `LinkedList` is not deprecated immediately — it remains valid for use cases where insert-by-reference or queue-like behavior is needed
- Consumers of `HTMLNode::children()` that type-hint `LinkedList` will need to update to `Vector` (breaking change in `webfiori/ui`, documented in migration guide)
