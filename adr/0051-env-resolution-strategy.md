# ADR-0051: Environment Variable Resolution Strategy

**Date:** 2026-09-21
**Status:** Accepted

## Context

The framework stores application constants (env vars) in a configuration driver
(JSON file, PHP class, or custom). At boot, `Controller::updateEnv()` reads these
values and registers them as PHP constants via `define()` and as process env vars
via `putenv()`.

The current resolution is simple: **the config driver value always wins**. There
is no mechanism for a system environment variable (e.g. one set by a CI/CD
pipeline, a container orchestrator, or the OS) to override a value stored in the
configuration.

This created a real problem in production: an application shipped a default
`app-config.json` with values like `GCP_PROJECT_ID = "webfiori"` as developer
defaults. On deployment to a CI/CD environment, the correct project ID was set as
a system env var (`GCP_PROJECT_ID=production-project`) — but the framework read
from the config file and ignored the system env, so the wrong value was used at
runtime.

The root cause is that the framework's model assumes **the config file is
authoritative**, which contradicts the widely-adopted
[12-factor app](https://12factor.net/config) principle that **the environment is
authoritative over committed configuration**.

This is a **feature addition**, not a bug fix. The `addEnvVar()` method and both
built-in drivers (`JsonDriver`, `ClassDriver`) work correctly. The gap is that
there is no way to express "system env should win at runtime" without writing
application code.

## Decision

Add an `EnvResolutionStrategy` enum and expose it via `App::setEnvResolutionStrategy()`
(which delegates to `Controller::setEnvResolutionStrategy()`), mirroring the
existing `App::setConfigDriver()` pattern.

### `EnvResolutionStrategy` enum

```php
enum EnvResolutionStrategy {
    /**
     * System environment variable wins. Config value is the fallback.
     * Correct for 12-factor / CI-CD deployments. This is the default.
     */
    case SYSTEM_FIRST;

    /**
     * Config value always wins. System env is never consulted.
     * Preserves pre-3.1 behavior for apps that need it.
     */
    case CONFIG_ONLY;

    /**
     * Only the system environment is consulted. Config value is ignored.
     * For locked-down server environments.
     */
    case SYSTEM_ONLY;
}
```

### Default: `SYSTEM_FIRST`

The default is `SYSTEM_FIRST`. This is a deliberate breaking change from the pre-3.1
behavior (`CONFIG_ONLY` was implicit). It ships alongside the v3.1 session system
changes. Apps that require the old behavior must opt-in explicitly:

```php
// In index.php or bootstrap, BEFORE App::init()
App::setEnvResolutionStrategy(EnvResolutionStrategy::CONFIG_ONLY);
App::init();
```

### API

```php
// App.php (public facade — mirrors setConfigDriver() pattern)
App::setEnvResolutionStrategy(EnvResolutionStrategy $strategy): void

// Controller.php (holds state, applies in updateEnv())
Controller::setEnvResolutionStrategy(EnvResolutionStrategy $strategy): void
Controller::getEnvResolutionStrategy(): EnvResolutionStrategy
```

### Resolution logic in `Controller::updateEnv()` (driver-agnostic)

The resolution strategy is applied in `Controller::updateEnv()` — the single
consumer of `getEnvVars()` from any driver. It works uniformly for `JsonDriver`,
`ClassDriver`, and any future or custom driver.

```
SYSTEM_FIRST:
  finalValue = getenv($name) ?: configValue
  (system env wins when non-empty; config is the fallback)

CONFIG_ONLY:
  finalValue = configValue
  (current/legacy behavior)

SYSTEM_ONLY:
  finalValue = getenv($name)
  (config value completely ignored; system env must be set)
```

### Configuration point

The strategy must be set **before `App::init()`** — the call that triggers
`Controller::updateEnv()`. Using a constant would create a circular dependency
(the constant would need to be `define()`'d by the same process that reads it),
so a static method call is used instead.

This is set once at application boot, typically in `index.php`.

## Alternatives Considered

**Per-variable `override` flag in config** — e.g. `{"value": "webfiori", "override": true}`.
Provides finer granularity, but is specific to the JSON driver format, breaks
the driver abstraction, and makes the config file responsible for runtime
resolution policy (a concern that belongs at the framework layer). Rejected.

**A `WF_ENV_RESOLUTION` PHP constant** — consistent with `WF_SESSION_STORAGE`
and `WF_VERBOSE`. However, this creates a chicken-and-egg problem: the constant
would need to be registered via `addEnvVar()` / `app-config.json`, but the
resolution strategy must be known before `updateEnv()` reads that file. Rejected.

**Always `SYSTEM_FIRST`, no configuration** — simpler, no enum needed. Rejected
in favor of flexibility: some deployments genuinely need `CONFIG_ONLY` (testing
environments, apps that manage config explicitly), and `SYSTEM_ONLY` is useful
for locked-down production servers.

**Keep `CONFIG_ONLY` as the default** — backward-compatible but perpetuates the
anti-pattern. Every new application would need to explicitly opt into the correct
modern behavior. Rejected: it is better to make the secure/correct behavior the
default and require explicit opt-out for the legacy behavior.

## Consequences

**Positive:**
- System environment variables (CI/CD, container, OS) work as expected without
  requiring deploy-time config manipulation.
- Aligns the framework with the 12-factor app model.
- Driver-agnostic: works for `JsonDriver`, `ClassDriver`, and all custom drivers.
- The config file can still hold developer defaults; they are just overridable
  at runtime.

**Negative / Migration:**
- **Breaking change from pre-3.1 behavior.** Any application where a system env
  var happens to be set with the same name as a config var will see the system
  env value instead of the config value after upgrading. This is almost always
  the intended behavior, but must be documented.
- Apps that require `CONFIG_ONLY` (e.g. strictly controlled config-file deploys)
  must add one line before `App::init()`.
- Ships as part of v3.1 alongside the session system changes.
