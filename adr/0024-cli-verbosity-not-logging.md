# ADR-0024: CLI Verbosity Levels Without Built-in Logging

**Date:** 2026-06-13
**Status:** Proposed

## Context

The `webfiori/cli` library provides output methods (`info()`, `error()`, `warning()`, `success()`, `println()`) that write unconditionally to the output stream. There is no way to:

- Suppress non-critical output for automated/cron environments (`--quiet`)
- Show additional diagnostic output for debugging (`-v`, `-vv`)

Enterprise CLI tools universally support verbosity flags. Without them, developers either flood logs with noise or miss important diagnostics.

A related question arose: should the CLI library also integrate structured logging (PSR-3 or `webfiori/log`)? This ADR documents the decision to add verbosity control but explicitly **not** add logging as a framework concern.

## Decision

**Add verbosity levels to `webfiori/cli`. Do not add logging.**

### Verbosity is the CLI library's responsibility

Verbosity controls *what the user sees in the terminal*. It is a presentation concern that belongs in the CLI framework:

```php
$this->info('Processing file...');     // Hidden in --quiet mode
$this->verbose('Opened 3 connections'); // Only shown with -v
$this->debug('Raw payload: ...');      // Only shown with -vv
$this->error('Failed to connect');     // Always shown
```

### Logging is the application's responsibility

Logging produces *structured records for machines* — timestamped, leveled, routed to files/services. The WebFiori ecosystem already has `webfiori/log` for this. Commands that need logging use it directly:

```php
use WebFiori\Log\Logger;

public function exec(): int {
    Logger::info('Command started', ['user' => $this->getArgValue('--user')]);
    // ...
}
```

The CLI library does not depend on, wrap, or re-export `webfiori/log`.

### Why these are separate concerns

| Aspect | Verbosity | Logging |
|--------|-----------|---------|
| Audience | Human at terminal | Ops/monitoring systems |
| Destination | stdout/stderr | Files, syslog, services |
| Format | Colored, formatted text | Structured (JSON, key=value) |
| Control | CLI flags (`-q`, `-v`) | Configuration/env vars |
| Library | `webfiori/cli` | `webfiori/log` |

## Alternatives Considered

### Add PSR-3 LoggerInterface as a dependency

Rejected because:
- Adds an external dependency (`psr/log`), violating the library's minimal-dependency philosophy.
- Conflates terminal output with structured logging — they serve different audiences.
- Forces users to configure a logger even for simple scripts that only need terminal output.
- `webfiori/log` already exists in the ecosystem for this purpose.

### Add `webfiori/log` as a hard or suggested dependency

Rejected because:
- Creates a circular concern — the CLI library should not know about logging implementation details.
- Users of `webfiori/cli` outside the WebFiori ecosystem would be forced into an opinionated logging choice.
- Violates separation of concerns: the CLI library is for terminal I/O, not application-level observability.

### Add a generic `setLogger()` hook on Runner

Rejected because:
- It implies the framework will call the logger at appropriate points, which requires the framework to decide *what* to log — that's an application decision.
- The existing `setAfterExecution()` hook already provides a place for users to wire logging if desired.
- A half-integrated logger (only logs command start/end) is worse than no logger — it gives false confidence.

## Consequences

### Benefits

- CLI output becomes controllable without code changes (just add `-q` or `-v`).
- Automated environments (cron, CI) can suppress noise with `--quiet`.
- Developers get richer diagnostics with `-v`/`-vv` without polluting normal output.
- Library stays dependency-free and focused on its core responsibility.
- Clear boundary: `webfiori/cli` handles terminal UX, `webfiori/log` handles observability.

### Trade-offs

- Developers who want both verbosity *and* logging must wire them separately. This is intentional — the two serve different audiences and should be configured independently.
- No automatic audit trail of command executions from the framework. Users who need this use `setAfterExecution()` to hook their own logger.

### Related

- ADR-0019: Decouple Framework Dependencies (same philosophy — libraries stay focused)
