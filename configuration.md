
# Configuration

<meta name="description" content="How WebFiori Framework configuration works: config drivers, app-config.json structure, config file composition with extends, write targets, and environment variable resolution.">

In this page:
* [Introduction](#introduction)
* [Config Drivers](#config-drivers)
* [app-config.json Structure](#app-configjson-structure)
* [Config File Composition](#config-file-composition)
  * [extends](#extends)
  * [Merge Strategies](#merge-strategies)
  * [inheritance_strategy](#inheritance_strategy)
  * [Write Targets](#write-targets)
  * [Cycle and Diamond Rules](#cycle-and-diamond-rules)
* [Environment Variable Resolution](#environment-variable-resolution)
* [Related Articles](#related-articles)

## Introduction

WebFiori stores application configuration in a **configuration driver** — a class that implements [`ConfigurationDriver`](https://webfiori.com/docs/WebFiori/Framework/Config/ConfigurationDriver). The driver is the single source of truth for settings like database connections, SMTP accounts, base URL, and named environment variables.

## Config Drivers

Two built-in drivers ship with the framework:

| Driver | Class | File | Best for |
|---|---|---|---|
| **JSON** (default) | `JsonDriver` | `App/Config/app-config.json` | Human-editable, CI/CD-friendly |
| **Class** | `ClassDriver` | `App/Config/AppConfig.php` | Performance (opcache-compiled) |

Switch the active driver in `index.php` before `App::init()`:

```php
use WebFiori\Framework\App;

App::setConfigDriver('\App\Config\AppConfig'); // use ClassDriver
App::init();
```

## app-config.json Structure

The JSON driver reads a single JSON file with these top-level keys:

```json
{
    "base-url": "https://example.com",
    "theme": null,
    "home-page": "BASE_URL",
    "primary-lang": "EN",
    "name-separator": "|",
    "scheduler-password": "NO_PASSWORD",
    "titles": { "EN": "My App" },
    "app-names": { "EN": "My App" },
    "app-descriptions": { "EN": "" },
    "version-info": { "version": "1.0", "version-type": "Stable", "release-date": "2024-01-01" },
    "env-vars": {
        "WF_VERBOSE": { "value": false, "description": "Show debug info" },
        "CLI_HTTP_HOST": { "value": "example.com", "description": "CLI host" }
    },
    "database-connections": {},
    "smtp-connections": {}
}
```

Use the CLI to add connections and env vars without editing the file manually:

```bash
php webfiori add:db-connection
php webfiori add:smtp-connection
```

## Config File Composition

> **Since 3.1** — `JsonDriver` only. See [ADR-0053](https://github.com/WebFiori/docs/blob/main/adr/0053-json-config-extends.md).

Large configs can be split into focused files and composed with `extends`. This is useful for:

- Keeping secrets (DB passwords, API keys) in a gitignored file while committing app metadata.
- Sharing a base config across multiple applications.
- Applying environment-specific overlays without duplicating the whole config.

### `extends`

`extends` is an array of file paths (relative to the extending file) to merge before applying the current file's values:

```json
{
    "extends": ["base.json", "db-connections.json", "smtp.json"],
    "base-url": "https://myapp.com"
}
```

**Child wins.** Values in the extending file override values in the extended files. Extended files are defaults/fallbacks.

**Left-to-right merge.** When multiple bases define the same key, later bases win. The child overrides all.

Paths can be simple filenames (sibling files) or relative paths:

```json
{
    "extends": ["../shared/org-base.json", "db-connections.json"]
}
```

### Merge Strategies

Different sections have different default merge behaviors:

| Section | Default | Behavior |
|---|---|---|
| `env-vars` | `merge` | All vars from all files; child overrides same key |
| `database-connections` | `merge` | All connections; child overrides same name |
| `smtp-connections` | `merge` | All accounts; child overrides same name |
| `app-names` | `merge` | Per-language, additive |
| `app-descriptions` | `merge` | Per-language, additive |
| `titles` | `merge` | Per-language, additive |
| `base-url` | `replace` | Child wins whole value |
| `theme` | `replace` | Child wins |
| `primary-lang` | `replace` | Child wins |
| `home-page` | `replace` | Child wins |
| `version-info` | `replace` | Child wins whole block |
| `scheduler-password` | `replace` | Child wins |

### `inheritance_strategy`

Override the default strategy for any section using `inheritance_strategy`:

```json
{
    "extends": ["base.json"],
    "inheritance_strategy": {
        "database-connections": "replace"
    },
    "database-connections": {
        "prod-db": { ... }
    }
}
```

With `replace`, only the child's `database-connections` survives — no merging with the base.

### Write Targets

When a CLI command writes to the config (e.g. `add:db-connection`), it writes to `app-config.json` by default. `write-targets` maps section names to the file that should receive writes for that section:

```json
{
    "extends": ["base.json", "db-connections.json", "smtp.json"],
    "write-targets": {
        "database-connections": "db-connections.json",
        "smtp-connections":     "smtp.json",
        "env-vars":             "env-vars.json"
    }
}
```

Now `php webfiori add:db-connection` writes to `db-connections.json` instead of `app-config.json`.

If the target file does not exist, it is created automatically with just the relevant section.

### Cycle and Diamond Rules

**Cycles throw.** If a file appears in its own ancestor chain, a `ConfigurationException` is thrown with the cycle path shown:

```
Circular config inheritance detected: a.json → b.json → a.json
```

**Diamonds are allowed.** If two bases both extend the same file, that shared base is resolved once and its result is reused — no duplicate processing.

```
app-config.json
  ├── b.json ──→ shared.json  ✓ resolved once
  └── c.json ──→ shared.json  ✓ reused (no re-read)
```

### Full example

```
App/Config/
├── app-config.json      (entry point, committed)
├── base.json            (app metadata, committed)
├── db-connections.json  (gitignored — contains DB passwords)
├── smtp.json            (gitignored — contains SMTP credentials)
└── env-vars.json        (gitignored — contains API keys)
```

```json
// app-config.json
{
    "extends": ["base.json", "db-connections.json", "smtp.json", "env-vars.json"],
    "write-targets": {
        "database-connections": "db-connections.json",
        "smtp-connections": "smtp.json",
        "env-vars": "env-vars.json"
    }
}
```

```json
// base.json
{
    "base-url": "https://myapp.com",
    "primary-lang": "EN",
    "version-info": { "version": "2.0", "version-type": "Stable", "release-date": "2026-01-01" }
}
```

```json
// db-connections.json  (gitignored)
{
    "database-connections": {
        "main": { "type": "mssql", "host": "prod-server", "database": "myapp", ... }
    }
}
```

## Environment Variable Resolution

> **Since 3.1**

By default (`SYSTEM_FIRST`), system environment variables take priority over values stored in `app-config.json`. This means a CI/CD pipeline can set `DB_HOST` as a system env var and the framework will use it automatically — no deploy-time config manipulation needed.

```php
// index.php — set BEFORE App::init()
use WebFiori\Framework\App;
use WebFiori\Framework\Config\EnvResolutionStrategy;

// Keep pre-3.1 behavior (config file wins):
App::setEnvResolutionStrategy(EnvResolutionStrategy::CONFIG_ONLY);

App::init();
```

| Strategy | Behavior |
|---|---|
| `SYSTEM_FIRST` (default) | System env wins; config is fallback |
| `CONFIG_ONLY` | Config always wins; system env ignored |
| `SYSTEM_ONLY` | Only system env; config ignored |

See [env-vars.md](env-vars) for the full list of recognized variables.

## Related Articles

* [Environment Variables](env-vars) — full reference of all recognized constants
* [Sessions Management](sessions-management) — configure session storage
* [Database Management](database) — add and use database connections
* [Sending Emails](sending-emails) — configure SMTP accounts
