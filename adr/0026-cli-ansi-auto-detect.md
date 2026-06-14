# ADR-0026: CLI ANSI Colors On by Default (TTY Auto-Detection)

**Date:** 2026-06-13
**Status:** Proposed

## Context

The library currently disables ANSI colored output by default. Users must pass `--ansi` explicitly to see colors:

```bash
php app.php greet         # no colors
php app.php greet --ansi  # colors
```

This is backwards from user expectations. Every modern CLI tool (git, npm, cargo, Symfony Console, Go CLI) shows colors when outputting to a terminal and suppresses them when piped or redirected.

The current behavior forces users to type `--ansi` every time or alias it — unnecessary friction for the common case.

## Decision

**ANSI output is enabled by default when stdout is a TTY, disabled when piped or redirected.**

### Resolution precedence (highest to lowest):

1. `--no-color` flag → OFF (explicit user override)
2. `NO_COLOR` environment variable set → OFF (respects [no-color.org](https://no-color.org) standard)
3. `--ansi` flag → ON (force colors even in piped context)
4. TTY auto-detection → ON if terminal, OFF if piped/redirected

### Detection logic:

```php
private static function shouldUseAnsi(): bool {
    if (getenv('NO_COLOR') !== false || isset($_SERVER['NO_COLOR'])) {
        return false;
    }
    if (function_exists('posix_isatty')) {
        return posix_isatty(STDOUT);
    }
    return false;
}
```

### `--ansi` flag is retained but its role changes:

- **Before:** opt-in to enable colors
- **After:** force colors in contexts where auto-detection would disable them (e.g., piping colored output to `less -R`)

### `--no-color` flag is added:

Allows users to explicitly disable colors in a terminal context (accessibility, preference, log capture).

## Alternatives Considered

### Keep colors off by default

Rejected because:
- Contradicts universal CLI convention
- Punishes the majority (terminal users) to protect the minority (piped contexts)
- Auto-detection solves both cases without user action

### Remove `--ansi` flag entirely

Rejected because:
- Still needed to force colors in piped contexts (`php app.php status --ansi | less -R`)
- Backward compatibility — existing scripts may use it

### Use `TERM` environment variable for detection

Rejected as primary mechanism because:
- `posix_isatty(STDOUT)` is more reliable and direct
- `TERM` can be set in non-interactive contexts (e.g., CI systems)
- Used only as fallback on Windows where `posix_isatty` is unavailable

## Consequences

### Benefits

- Colors work out of the box — better first-time experience
- Piped output is clean automatically — no garbage ANSI codes in files
- Respects `NO_COLOR` standard (accessibility)
- Aligns with every major CLI tool's behavior

### Trade-offs

- Behavioral change: terminal output that was previously uncolored now has colors. This is intentionally a better default, but could surprise users who depend on the previous plain output in terminals.
- Users who explicitly want no colors in terminals must pass `--no-color` or set `NO_COLOR=1`.
- `posix_isatty` is not available on Windows without extensions; Windows falls back to environment variable checks (`ANSICON`, `ConEmuANSI`).

### Related

- ADR-0024: CLI Verbosity (another output behavior enhancement)
- [no-color.org](https://no-color.org) — the `NO_COLOR` convention already respected by `Formatter`
