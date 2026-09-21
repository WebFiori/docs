# ADR-0053: JsonDriver Config File Composition via `extends`

**Date:** 2026-09-21
**Status:** Accepted

## Context

As applications grow, a monolithic `app-config.json` becomes hard to manage.
Database connections, SMTP accounts, environment variables, and app metadata are
unrelated concerns that have no reason to share a single file. Teams also want to:

- Keep secrets (DB passwords, API keys) in a gitignored file while committing
  app metadata.
- Share a base config across multiple applications in an organization.
- Apply environment-specific overlays (staging vs production) without duplicating
  the whole config.

The `ClassDriver` does not have this problem — PHP class inheritance already
handles composition. This is a `JsonDriver`-specific need.

## Decision

Add an `"extends"` key to `app-config.json` (and any JSON config file) that
lists one or more base config files to merge before applying the current file's
values. The resolution is handled by a new `JsonConfigInheritance` class,
modeled on the `AgentProfile` inheritance system (ADR-0045).

### `extends` syntax

```json
{
  "extends": ["base.json", "db-connections.json", "smtp.json"],
  "inheritance_strategy": {
    "database-connections": "replace"
  },
  "write-targets": {
    "database-connections": "db-connections.json",
    "smtp-connections":     "smtp.json",
    "env-vars":             "env-vars.json"
  },
  "base-url": "https://myapp.com"
}
```

`extends`, `inheritance_strategy`, and `write-targets` are **resolution-time
directives** — consumed during loading and stripped from the merged result.
They never appear in `getEnvVars()`, `getDBConnections()`, etc.

### Resolution rules

**1. Child wins.** Values in the extending file override values in extended files.
Extended files are defaults/fallbacks.

**2. Array of bases, resolved left-to-right.** Each base is merged in order;
later bases override earlier ones; child overrides all.

**3. Per-section merge strategy.** Different sections have different default
strategies, overridable via `inheritance_strategy`:

| Section | Default | Reason |
|---|---|---|
| `env-vars` | `merge` | Additive; child overrides same key |
| `database-connections` | `merge` | All connections; child overrides same name |
| `smtp-connections` | `merge` | All accounts; child overrides same name |
| `app-names` | `merge` | Per-language, additive |
| `app-descriptions` | `merge` | Per-language, additive |
| `base-url` | `replace` | Scalar — child wins whole |
| `theme` | `replace` | Scalar |
| `primary-lang` | `replace` | Scalar |
| `name-separator` | `replace` | Scalar |
| `home-page` | `replace` | Scalar |
| `scheduler-password` | `replace` | Scalar (should be system env) |
| `version-info` | `replace` | Whole block |

**4. Cycles throw, diamonds are allowed.** A file appearing in its own ancestor
chain throws `ConfigurationException` with the cycle path. A file appearing in
multiple branches (diamond) is resolved once; its result is reused.

**5. Paths are relative to the extending file.** Consistent with standard
include/import semantics; config directories are self-contained.

### `write-targets`

When a CLI command writes to the config (e.g. `add:db-connection`), it writes
to the entry point by default. `write-targets` maps section names to the file
that should receive writes for that section.

```json
"write-targets": {
  "database-connections": "db-connections.json"
}
```

If the target file does not exist, it is created with just the relevant section.
If the target file already exists, only its section is updated; other sections
are untouched.

`write-targets` is supported via a new `getWriteTarget(string $section): string`
method on `ConfigurationDriver` — returning the entry point path by default,
the target path when configured. CLI commands can use this to report where a
value was written.

### `JsonConfigInheritance` class

An instance class (one instance per `initialize()` call) that handles the full
resolution pipeline:

- Instance state: `$visited` (current DFS stack for cycle detection),
  `$resolved` (path → merged array, for diamond deduplication).
- `resolve(string $entryPath): array` — public entry point.
- `resolveFile(string $path, array $childData = null): array` — recursive.
- Returns the fully merged array with directives stripped.

### File-driver convention

`extends` is a `JsonDriver`-specific feature. It is not added to the
`ConfigurationDriver` interface. Future file-based drivers (`.env`, YAML) may
adopt the same convention; compiled or DB-backed drivers handle composition via
their own native mechanisms (PHP class inheritance, DB joins, etc.).

## Alternatives Considered

**Static class for `JsonConfigInheritance`.** Rejected — static state leaks
between tests and re-initializations. An instance class is testable and
consistent with the `AgentProfile` implementation.

**Single-string `extends` (one parent only).** Rejected — the primary use case
is composing independent concern files (db, smtp, env-vars). A linear chain
would force artificial ordering and add extra files. Array extends is strictly
more capable for no implementation cost.

**Putting `write-targets` in code, not in the file.** Rejected — keeping all
config topology in the config file means no code changes are needed when
restructuring. The config file is self-describing.

**Adding `extends` to `ConfigurationDriver` interface.** Rejected — this is a
file-based concern. The interface stays lean and driver-agnostic.

## Consequences

**Positive:**
- Large config files split into maintainable, concern-specific partials.
- Secrets can live in gitignored files; app metadata in committed files.
- Shared base configs across multiple applications in an organization.
- Environment overlays without duplicating the whole config.
- Diamond resolution prevents redundant base-file processing.

**Negative:**
- Resolution adds file I/O at boot (one read per extended file). Mitigated by
  opcache and the fact that `ClassDriver` (which benefits from full opcache) does
  not need this mechanism.
- Developers must understand the merge semantics and `inheritance_strategy`.
  Good documentation and clear error messages (cycle paths) mitigate this.
- `write-targets` adds a layer of indirection to CLI commands. The default
  (write to entry point) means existing usage is unchanged.
