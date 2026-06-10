# ADR-0020: Backward Compatibility Guarantee

**Date:** 2026-06-02
**Status:** Proposed

## Context

The enterprise readiness plan introduces significant new capabilities (streaming, interfaces, emitters, file locking, security fixes). Several of these could be implemented as breaking changes:

- `getSize()` returning `?int` (null) instead of `int` (-1)
- Removing `die()` from `view()`
- Removing framework imports from `File.php`
- Renaming `$extentions` to `$extensions`

A major version bump (v3) would allow all of these cleanly but imposes migration cost on every existing user. The library has active consumers (visible from Packagist downloads). Forcing migration without strong justification harms trust and adoption.

## Decision

**All changes ship as backward-compatible minor releases (v2.x).** No v3 is needed. Each potentially-breaking change is resolved with a BC-compatible alternative:

### `getSize()` — keep returning `int`

```php
// Unchanged
public function getSize(): int {
    return $this->fileSize; // remains -1 when unknown
}

// New companion method
public function hasKnownSize(): bool {
    return $this->fileSize >= 0;
}
```

Existing code checking `getSize() === -1` continues to work. New code uses `hasKnownSize()`.

### `die()` in `view()` — keep it, add opt-out

```php
public function view(bool $asAttachment = false, bool $terminate = true): void {
    // ... existing logic ...
    if ($terminate && http_response_code() !== false) {
        die();
    }
}
```

Default behavior unchanged. Users pass `terminate: false` to opt out. The streaming API (`FileStream::serve()`) never terminates.

### Framework imports — conditional loading preserved

`File::view()` keeps its current `class_exists()` check internally. No behavioral change for existing users. The new streaming API uses emitters and has no framework imports.

Over time, `view()` can be soft-deprecated in documentation pointing users toward `stream()->serve()`.

### `$extentions` typo — already private

Private properties are not part of the public API. Renaming is not a BC break.

### New methods on existing classes

Adding methods (`stream()`, `hasKnownSize()`, `copy()`, `moveTo()`) to `File` is not a BC break. Adding interfaces (`FileInterface`) that `File` implements is not a BC break — existing type hints of `File` still work.

### Deprecation strategy

Methods that have better alternatives get `@deprecated` annotations pointing to the replacement:

```php
/**
 * @deprecated Use FileStream::serve() for memory-efficient serving.
 * @see FileStream::serve()
 */
public function view(bool $asAttachment = false, bool $terminate = true): void
```

Deprecated methods are removed only in a future major version (if ever needed).

## Alternatives Considered

- **Ship as v3.0.0 with clean breaks** — rejected because the breaks are minor and avoidable. A major version signals "you must migrate" — there's nothing here that justifies that signal.
- **Use `@trigger_error` deprecation notices** — considered for `view()` but rejected as too aggressive. A docblock annotation is sufficient. Runtime deprecation notices can be added later if usage doesn't decline.
- **Branch maintenance (v2.x + v3.x)** — rejected because maintaining two branches doubles workload for a single-maintainer project.

## Consequences

### Benefits

- Zero migration cost for existing users. `composer update` just works.
- New features are opt-in through new methods/classes.
- Builds trust: users know upgrading within v2.x won't break their code.
- Simplifies release process: no branching strategy needed.

### Trade-offs

- Some APIs remain "messy" (`getSize()` returning -1, `view()` with `die()`). They work, but the newer alternatives are cleaner.
- Documentation must clearly guide users toward the new APIs without making the old ones feel broken.
- The `-1` sentinel and `die()` can never be removed without a v3.

### Related

- ADR-0016: FileInterface (additive, non-breaking)
- ADR-0017: Streaming (new class, non-breaking)
- ADR-0018: ResponseEmitter (new interface, non-breaking)
- ADR-0019: Framework decoupling (preserves File::view() behavior)
