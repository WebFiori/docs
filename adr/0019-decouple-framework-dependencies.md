# ADR-0019: Decouple webfiori/file from Framework Dependencies

**Date:** 2026-06-02
**Status:** Proposed

## Context

`File.php` contains these imports:

```php
use WebFiori\Framework\App;
use WebFiori\Http\Response;
```

And conditional logic:

```php
if (class_exists('\WebFiori\Http\Response')) {
    $this->useClassResponse($contentType, $asAttachment);
} else {
    $this->doNotUseClassResponse($contentType, $asAttachment);
}
```

Meanwhile, `composer.json` lists these as dev dependencies:

```json
"require-dev": {
    "webfiori/http": "*",
    "webfiori/framework": "dev-dev"
}
```

Problems:
- A file library should not know about HTTP response classes or application frameworks.
- The wildcard `"*"` and `"dev-dev"` constraints are unstable for CI reproducibility.
- Runtime `class_exists()` makes behavior depend on what packages happen to be installed — different behavior in dev vs production if `require-dev` packages are not installed.
- Users of other frameworks cannot cleanly integrate because serving logic is hardcoded to WebFiori's Response class.

## Decision

1. **Remove `use WebFiori\Framework\App` and `use WebFiori\Http\Response`** from `File.php`.
2. **Move `webfiori/http` and `webfiori/framework`** to `suggest` in `composer.json`.
3. **Pin dev dependencies** to stable version ranges.
4. **Replace `class_exists()` conditional** with the `ResponseEmitter` pattern (ADR-0018).
5. **Ship `WebFioriEmitter`** as an optional adapter class that users import only if they use WebFiori's HTTP layer.

### composer.json changes

```json
{
    "require": {
        "php": ">=8.1",
        "webfiori/jsonx": "^4.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "webfiori/http": "^3.0",
        "webfiori/framework": "^4.0"
    },
    "suggest": {
        "webfiori/http": "Required for WebFioriEmitter - serves files using WebFiori's Response class"
    }
}
```

### What happens to `File::view()`

For backward compatibility, `File::view()` remains and keeps its behavior. Internally, it uses `DefaultEmitter` (raw headers + echo). The old `useClassResponse()` path is removed from `File` and lives only in `WebFioriEmitter`.

Users who need the WebFiori Response integration use:

```php
$file->stream()->serve(emitter: new WebFioriEmitter());
```

Or, for one-shot:

```php
$file->view(); // uses DefaultEmitter internally, no framework dependency
```

### File structure after decoupling

```
WebFiori/File/
├── File.php                  (no framework imports)
├── FileStream.php
├── FileInterface.php
├── StreamableInterface.php
├── ResponseEmitter.php       (interface)
├── DefaultEmitter.php        (raw header/echo)
├── WebFioriEmitter.php       (optional, uses webfiori/http)
├── FileUploader.php
├── UploadedFile.php
├── MIME.php
├── UploaderConst.php
└── Exceptions/
    └── FileException.php
```

## Alternatives Considered

- **Keep conditional `class_exists()` logic** — rejected because it makes behavior unpredictable and is a maintenance burden. Two code paths that diverge silently is a bug factory.
- **Remove HTTP serving entirely from the library** — rejected because file serving with proper headers is a legitimate core feature. The issue is not that the library serves files, it's that it's coupled to a specific framework's response class.
- **Make `webfiori/http` a hard requirement** — rejected because it forces every user to install WebFiori's HTTP layer even if they use Symfony or Laravel.
- **Create a separate `webfiori/file-http` package for serving** — rejected as over-engineering. A single `ResponseEmitter` interface within the same package is sufficient.

## Consequences

### Benefits

- `webfiori/file` becomes a true standalone library — installable and usable without any framework.
- No more implicit behavior differences between environments.
- Cleaner dependency tree for consumers.
- Stable CI builds with pinned dev dependencies.

### Trade-offs

- Users who rely on `File::view()` auto-detecting WebFiori's Response class will need to switch to `FileStream::serve(emitter: new WebFioriEmitter())` if they want Response-object integration. However, `File::view()` continues to work — it just uses raw headers instead of the Response class.
- `WebFioriEmitter` class must be maintained alongside the main library if `webfiori/http` introduces breaking changes.

### Related

- ADR-0017: Streaming (FileStream is the new primary serving mechanism)
- ADR-0018: ResponseEmitter (the abstraction that enables decoupling)
- ADR-0020: Backward compatibility (explains why File::view() is not removed)
