# ADR-0052: Configuration System Design Review

**Date:** 2026-09-21
**Status:** Proposed — direction for v4

## Context

This ADR is a **design review**, not a decision to change things immediately.
It records the findings of a deliberate review of the configuration system,
identifies design problems, and proposes the correct long-term model so that
future contributors work toward it rather than away from it.

### How the current system works

The framework stores application configuration in a driver that implements
`ConfigurationDriver`. Two built-in drivers exist:

- **`JsonDriver`** — reads and writes `app-config.json`. Default driver.
  Human-editable, easy to inspect.
- **`ClassDriver`** — compiles configuration into a PHP class, loaded via the
  autoloader. Faster than file I/O; benefits from opcache.

The `env-vars` section of the config holds named entries. At boot,
`Controller::updateEnv()` iterates these and for each entry:

1. Calls `define($name, $value)` — registering the value as a PHP constant.
2. Calls `putenv("$name=$value")` — also registering it as a process env var.

Application code then reads these values as **bare PHP constants**:

```php
// Framework internals
$sessionKey = defined('SESSION_KEY') ? SESSION_KEY : null;
if (defined('WF_VERBOSE') && WF_VERBOSE === true) { ... }
$host = defined('CLI_HTTP_HOST') ? CLI_HTTP_HOST : '127.0.0.1';
```

A bespoke `env:VARNAME` prefix syntax inside the config file allows a value to
reference a real system environment variable:

```json
"SESSION_KEY": { "value": "env:SESSION_KEY", "description": "..." }
```

### What is correct about the design

1. **Multi-driver extensibility is a deliberate, good design principle.** The
   `ConfigurationDriver` interface enables JSON, PHP class, database, `.env`,
   secrets-manager, and any other backend without changing the framework core.
   This must be preserved.

2. **`ClassDriver` serves a legitimate performance purpose.** Compiled PHP in
   opcache is faster than reading a JSON file on every request. For high-traffic
   applications, `ClassDriver` is a valid choice. It remains relevant even after
   the problems below are fixed.

3. **`putenv()` is correct.** Making config values available as process env vars
   via `putenv()` is the right mechanism — it integrates with `getenv()`, which
   is the standard cross-framework way to read runtime configuration.

### What is wrong

**Problem 1 — `define()` conflates two unrelated concepts.**

`env-vars` holds two fundamentally different kinds of values:

| Kind | Example | Right home |
|---|---|---|
| Runtime env var | `DB_HOST`, `GCP_PROJECT_ID`, `SESSION_KEY` | System env / config fallback → `getenv()` |
| True code constant | `MAX_RETRY = 3`, `DEFAULT_LANGUAGE = 'EN'` | PHP class constant or `define()` in bootstrap |

The current design puts both in `env-vars` and `define()`s everything. This is
wrong: runtime env vars are not code constants. They change between deployments,
they should come from the environment, and they must be testable (replaceable
with `putenv()` in tests — but `define()` is immutable once set).

**Problem 2 — `define()` makes testing painful.**

Once `WF_VERBOSE` is `define()`'d as `false`, no test in the same process can
change it. The framework works around this with `if (!defined('WF_VERBOSE'))` guards
everywhere, which is defensive code masking a design problem.

**Problem 3 — bare constants obscure the source.**

Reading `SESSION_KEY` as a bare constant looks like a compile-time code constant.
A developer joining the project has no indication it comes from runtime configuration
until they trace the `define()` call in `Controller::updateEnv()`. `getenv('SESSION_KEY')`
is self-documenting.

**Problem 4 — secrets like `SESSION_KEY` must not be in config files.**

Encryption keys are secrets. They should come exclusively from the system
environment or a secrets manager — never from a committed or even deployed config
file. The current design encourages storing `SESSION_KEY` in `app-config.json`
(even if as `"value": "env:SESSION_KEY"`), which normalises the wrong mental model.

**Problem 5 — the `env:VARNAME` prefix is bespoke and undiscoverable.**

No other framework uses this notation. Developers unfamiliar with WebFiori have
no way to know that `"host": "env:DB_HOST"` in a JSON file means "read from
system env at runtime." It is a custom DSL where a standard mechanism (`getenv()`)
already exists.

## Decision

### v3.x (deprecation, no removal)

- **Deprecate `define()` in `Controller::updateEnv()`.**
  The method continues to call both `putenv()` and `define()` for backward
  compatibility, but `define()` is marked deprecated. Framework internals are
  progressively updated to use `getenv()` instead of bare constants.
- The `env:VARNAME` prefix remains supported but is documented as a legacy
  mechanism. `EnvResolutionStrategy::SYSTEM_FIRST` (ADR-0051) provides the
  clean replacement: set the value in system env, let the framework's resolution
  strategy pick it up, read via `getenv()`.

### v4 (removal and clean model)

1. **`define()` is removed from `Controller::updateEnv()`.**
   All `env-vars` values are registered with `putenv()` only.

2. **Framework internals switch from bare constants to `getenv()`.**
   ```php
   // Before (v3)
   $key = defined('SESSION_KEY') ? SESSION_KEY : null;

   // After (v4)
   $key = getenv('SESSION_KEY') ?: null;
   ```

3. **True code constants leave the config system.**
   Values that are genuinely fixed for the codebase (not the environment) belong
   in PHP class constants or a dedicated bootstrap file — not in `app-config.json`.
   The `env-vars` section is for runtime/deployment values only.

4. **`env:VARNAME` prefix is removed.**
   With `SYSTEM_FIRST` as the default resolution strategy (ADR-0051), there is no
   need for an in-file reference syntax. System env vars are consulted automatically.
   The prefix is deprecated in v3.x and removed in v4.

5. **Multi-driver model is preserved and extended.**
   `ConfigurationDriver` remains the abstraction. Built-in drivers (`JsonDriver`,
   `ClassDriver`) remain. Future drivers (database, `.env`, vault) follow the same
   interface. `ClassDriver`'s performance advantage is unchanged — opcache still
   benefits, and values are still available via `getenv()` after `putenv()`.

6. **Secrets guidance is added to documentation.**
   Encryption keys and other secrets must come from system env only. The config
   file should not hold or reference them — not even via `env:` prefix.

### The correct mental model (v4)

```
┌─────────────────────────────────────────────────────────────┐
│                   Configuration Sources                     │
├─────────────────────────────────────────────────────────────┤
│  System env    │  Highest priority (CI/CD, containers, OS)  │
│  Config driver │  Fallback defaults (JsonDriver, ClassDriver │
│                │  DB driver, .env driver, ...)               │
├─────────────────────────────────────────────────────────────┤
│                   Access Pattern                            │
├─────────────────────────────────────────────────────────────┤
│  Runtime values │  getenv('DB_HOST'), getenv('SESSION_KEY') │
│  Code constants │  MyClass::CONSTANT, define() in bootstrap  │
└─────────────────────────────────────────────────────────────┘
```

## Alternatives Considered

**Keep `define()` permanently, fix only testing ergonomics.**
Rejected. Bare constants obscure the source, making code harder to understand
and onboard. The right fix is the standard mechanism, not better workarounds.

**Remove `ClassDriver`.**
Rejected. It has a legitimate performance justification and fits cleanly in the
multi-driver model. Its `define()` behavior is the problem, not its existence.

**Replace `env:VARNAME` with a richer template syntax.**
Rejected. Adding more bespoke syntax makes the problem worse. `SYSTEM_FIRST`
resolution (ADR-0051) eliminates the need for in-file variable references.

**Add a `type: constant` marker to distinguish true constants from env vars.**
Rejected. True code constants do not belong in the config system at all — they
belong in PHP code. A marker is a workaround for the wrong design.

## Consequences

**Positive:**
- Clear mental model: config = runtime values, accessed via `getenv()`.
- `define()` removed: testing is straightforward, no immutability traps.
- Secrets guidance: encryption keys come from system env only.
- Multi-driver extensibility preserved: DB driver, `.env` driver, vault driver
  can be added without changing the framework core.
- Aligns with how every other major PHP framework (Laravel, Symfony) works.

**Negative / Migration:**
- v4 is a breaking change for any application code that reads config values as
  bare PHP constants. Migration: replace `MY_CONSTANT` with `getenv('MY_CONSTANT')`.
- Applications storing true code constants in `app-config.json` must move them
  to PHP files.
- The `env:VARNAME` prefix must be removed from config files (can be automated
  with a migration command).

## Open Questions (for v4 planning)

1. Should a migration CLI command be provided to scan `app-config.json` for
   bare constant usages in application code and suggest `getenv()` replacements?
2. Should `getenv()` be wrapped in a framework helper (e.g. `App::env('KEY', $default)`)
   for a consistent access pattern and easier future changes (e.g. switching to
   `$_ENV` or a DI container)?
3. How should `ClassDriver` handle the transition — generate `putenv()` calls
   instead of `define()` in the compiled PHP class?
