# ADR-0013: Deprecate Global Constants in Favor of Static Defaults

**Date:** 2026-06-02
**Status:** Proposed

## Context

The library uses PHP global constants for configuration:

```php
if (defined('JSON_STYLE')) {
    $propsStyle = JSON_STYLE;
}

if (defined('JSON_CASE')) {
    $lettersCase = JSON_CASE;
}

$this->setIsFormatted($isFormatted === true || (defined('WF_VERBOSE') && WF_VERBOSE === true));
```

Global constants have these problems:

1. **Not testable in isolation** — once defined, a constant cannot be undefined or changed. Tests that need different styles must run in separate processes.
2. **Not composable** — you cannot have one part of an application use `camel` and another use `snake` if the global is set.
3. **Framework coupling** — `WF_VERBOSE` ties this standalone library to the WebFiori framework. A library consumed via Composer should be framework-agnostic.

## Decision

Deprecate `JSON_STYLE`, `JSON_CASE`, and `WF_VERBOSE` global constants. Replace with a static defaults method:

```php
Json::setDefaults(style: 'camel', case: 'lower', formatted: false);
```

Behavior:
- `Json::setDefaults()` sets application-wide defaults.
- Constructor parameters override defaults per-instance.
- Global constants are still checked (for backward compatibility) but trigger a deprecation notice.
- In the next major version, global constant support is removed entirely.

```php
// Application bootstrap
Json::setDefaults(style: 'camel', case: 'lower');

// Uses defaults: camel + lower
$json1 = new Json(['first-name' => 'Ibrahim']);

// Overrides per-instance: snake + upper
$json2 = new Json(['first-name' => 'Ibrahim'], 'snake', 'upper');
```

## Alternatives Considered

- **Configuration object (`JsonConfig`)** — more structured but heavier for a simple library with only 3 settings.
- **Environment variables** — same testability problems as constants, plus not type-safe.
- **Keep constants, document the limitation** — does not solve testability or composability.

## Consequences

- **Backward compatible during deprecation phase**: existing code using constants continues to work with a deprecation notice.
- **Testable**: `Json::setDefaults()` can be called in `setUp()` and reset in `tearDown()`.
- **Composable**: per-instance overrides work independently of global defaults.
- **Framework decoupled**: no reference to `WF_VERBOSE` or any framework-specific constant.
- **Migration path**: deprecation notice in current major version, removal in next.

## GitHub Issue

[webfiori/json#61](https://github.com/WebFiori/json/issues/61)
