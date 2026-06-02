# ADR-0017: Extract Renderer from HTMLNode

**Date:** 2026-06-03
**Status:** Proposed

## Context

`HTMLNode` is a 2800+ line class that combines DOM structure management with HTML/XML rendering and code highlighting. The rendering logic uses instance variables (`$htmlString`, `$nodesStack`, `$tabCount`, `$tabSpace`, `$nl`) as temporary state during serialization. This causes several problems:

1. **Non-reentrant rendering**: If `toHTML()` is called on a node while another `toHTML()` is in progress (e.g., via `__toString()` triggered during rendering), the shared instance state is corrupted.
2. **Untestable in isolation**: Rendering behavior cannot be tested without constructing full node trees.
3. **Static state leakage**: `$IsFormatted`, `$IsQuoted`, `$UseForwardSlash` are static variables that affect all instances globally, making the class unsafe for concurrent use in async runtimes (Swoole, ReactPHP, RoadRunner).
4. **Single Responsibility violation**: The class handles tree structure, attribute management, iteration, factory methods, and three different serialization formats.

## Decision

Extract rendering logic into a dedicated `HtmlRenderer` class. Keep the existing public API on `HTMLNode` intact by delegating to the renderer internally.

**New class:**

```php
namespace WebFiori\Ui;

class HtmlRenderer {
    private string $output = '';
    private int $tabCount = 0;
    private string $tabSpace = '';
    private string $nl = '';
    private bool $formatted;
    private bool $quoted;
    private bool $useForwardSlash;

    public function __construct(bool $formatted = false, bool $quoted = false, bool $useForwardSlash = false) {
        $this->formatted = $formatted;
        $this->quoted = $quoted;
        $this->useForwardSlash = $useForwardSlash;
    }

    public function render(HTMLNode $node, int $initTab = 0): string {
        // All rendering logic moved here
        // Uses local state only — fully reentrant
    }
}
```

**HTMLNode delegation (backward-compatible):**

```php
public function toHTML(bool $formatted = false, int $initTab = 0): string {
    $renderer = new HtmlRenderer($formatted, self::$IsQuoted, self::$UseForwardSlash);
    return $renderer->render($this, $initTab);
}

public function toXML(bool $formatted = false): string {
    $renderer = new HtmlRenderer($formatted, true, true);
    return '<?xml version="1.0" encoding="UTF-8"?>' 
         . ($formatted ? HTMLDoc::NL : '') 
         . $renderer->render($this);
}
```

The `asCode()` method follows the same pattern with a `CodeRenderer` class.

Static methods `setIsFormatted()`, `setIsQuotedAttribute()`, `setUseForwardSlash()` remain for backward compatibility but are deprecated. The renderer constructor is the new way to control output format.

## Alternatives Considered

1. **Keep everything in HTMLNode but use local variables**: Would reduce the instance-state problem but doesn't address testability, SRP, or the 2800-line file size.

2. **Make rendering a trait**: Traits can't have their own state cleanly and don't improve testability.

3. **Break backward compatibility and remove toHTML() from HTMLNode**: Too disruptive. Users call `$node->toHTML()` everywhere.

## Consequences

**Easier:**
- Rendering is reentrant and safe for async contexts
- Renderer can be unit-tested independently
- Future serialization formats (JSON, Markdown) follow the same extraction pattern
- `HTMLNode` shrinks by ~400 lines
- Static state can be deprecated over time

**Harder:**
- Two places to look for rendering logic during transition
- Slight overhead from object creation (negligible — one small object per `toHTML()` call)

**Backward compatibility:** Fully preserved. All existing `toHTML()`, `toXML()`, `asCode()` calls continue to work unchanged.
