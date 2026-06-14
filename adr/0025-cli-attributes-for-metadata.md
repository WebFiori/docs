# ADR-0025: CLI Command Metadata via PHP Attributes

**Date:** 2026-06-13
**Status:** Accepted

## Context

The `webfiori/cli` library currently has two ways to attach metadata to commands:

1. **Constructor parameters** — name, description, arguments passed to `parent::__construct()`
2. **Docblock annotations** — `@Command(group="db", aliases=["m"], hidden=true)` parsed by `CommandMetadata::extract()` using regex

Problems with docblock annotations:
- Not statically analyzable (IDEs and static analysis tools can't validate them)
- Fragile regex parsing — breaks on formatting variations
- No autocomplete or type safety
- Not a standard PHP mechanism — custom convention with no tooling support
- Only work with auto-discovery (`CommandDiscovery`), not manual `register()` calls

PHP 8.1 native attributes solve all of these. Since the library already requires PHP 8.1+, there is no compatibility barrier.

## Decision

**Adopt PHP 8.1 attributes as the declarative metadata mechanism for commands. Deprecate docblock annotation parsing.**

### Attribute classes (under `WebFiori\Cli\Attributes\`):

```php
#[Attribute(Attribute::TARGET_CLASS)]
class Group {
    public function __construct(public readonly string $name) {}
}
```

Future attributes (not in scope now, but the pattern supports them):
```php
#[Attribute(Attribute::TARGET_CLASS)]
class Hidden {}

#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
class Alias {
    public function __construct(public readonly string $name) {}
}
```

### Resolution precedence (highest to lowest):

1. **Explicit setter** — `$command->setGroup('db')` (programmatic, always wins)
2. **PHP attribute** — `#[Group('db')]` on the class (declarative)
3. **Convention** — colon in command name (`db:migrate` → group `db`)

This precedence applies to all metadata: group, hidden, aliases, etc.

### Attributes are always optional

The programmatic API (`setGroup()`, `setAliases()`, constructor params) remains the primary way to configure commands. Attributes are syntactic sugar for the common case. A command with no attributes works identically to today.

### Command name is the invocation identifier

The command name (first constructor parameter) is exactly what the user types. Group is purely organizational for help display. A command named `db:migrate` is invoked as `php app.php db:migrate` — the group does not affect resolution or add implicit prefixes.

### Deprecation of docblock parsing

`CommandMetadata::extract()` currently parses `@Command(...)` from docblocks. This will be:
1. Maintained for one minor version (backward compat)
2. Marked `@deprecated` with guidance to use attributes
3. Removed in the next major version

## Alternatives Considered

### Keep docblock annotations

Rejected because:
- No IDE support, no static analysis, fragile parsing
- Dual systems (docblock + attributes) would confuse users
- PHP has a native mechanism now — no reason to maintain a custom one

### Use attributes exclusively with no setter fallback

Rejected because:
- Breaks existing code that uses `setGroup()` or passes metadata in constructors
- Makes programmatic/dynamic command configuration impossible
- Attributes can't be added at runtime (e.g., commands created from config files)

### Derive all metadata from command name conventions only

Rejected because:
- Forces naming conventions that may not fit every project
- Can't express "hidden" or arbitrary grouping through name alone
- Implicit magic — harder to understand than explicit declaration

## Consequences

### Benefits

- IDE autocomplete and validation for command metadata
- Static analysis tools (PHPStan, Psalm) can verify attribute usage
- Single consistent pattern for all future metadata needs
- Clean deprecation path for fragile docblock parsing
- Attributes are self-documenting — visible on the class declaration

### Trade-offs

- `CommandDiscovery` and `Runner::register()` now need reflection to read attributes (minor performance cost at registration time, not at runtime)
- Deprecation period means maintaining both docblock and attribute reading temporarily
- Users on PHP 8.1 already have attribute support, but some may be unfamiliar with the syntax

### Related

- ADR-0019: Decouple Framework Dependencies (same philosophy — use standard PHP mechanisms)
- ADR-0024: CLI Verbosity (another CLI library enhancement in the same release)

## Implementation

Implemented in `webfiori/cli` PR #55 (branch `feat/command-attributes`):

### Delivered attributes:

| Attribute | Purpose | Location |
|-----------|---------|----------|
| `#[Group('name')]` | Organizes commands in help output | `WebFiori\Cli\Attributes\Group` |
| `#[SingleInstance]` | Prevents concurrent execution via flock | `WebFiori\Cli\Attributes\SingleInstance` |

### Resolution precedence (as designed):

1. **Explicit setter** — `$command->setGroup('db')` (programmatic, always wins)
2. **PHP attribute** — `#[Group('db')]` on the class
3. **Convention** — colon in command name (`db:migrate` → group `db`)

### Supporting classes:

- `WebFiori\Cli\LockManager` — flock-based non-blocking exclusive locks with PID tracking
- `Command::resolveGroup()` — resolves group from attribute/convention at registration time
- `Command::resolveSingleInstance()` — reads attribute via reflection before `exec()`
- `HelpCommand::printCommandsGrouped()` — grouped help display with alphabetical group sorting

### Lock lifecycle:

- Acquired before `exec()` if `#[SingleInstance]` present
- Released in `finally` block (even on exceptions)
- OS releases flock on process crash/SIGKILL (file handle closed)
