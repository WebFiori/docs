# ADR-0018: ResponseEmitter Strategy for HTTP File Serving

**Date:** 2026-06-02
**Status:** Proposed

## Context

The `File::view()` method currently has two code paths chosen at runtime:

1. If `WebFiori\Http\Response` class exists → use it to build a response object.
2. Otherwise → use raw `header()` + `echo` + `die()`.

This creates several problems:

- **Hidden coupling**: `File.php` imports `WebFiori\Framework\App` and `WebFiori\Http\Response`. A "file" library should not know about HTTP frameworks.
- **Runtime class detection**: `class_exists()` checks make behavior unpredictable — the same code behaves differently depending on what packages are installed.
- **Untestable**: the raw-header path calls `die()`, making it impossible to test. The Response path checks for `__PHPUNIT_PHAR__` — test concerns in production code.
- **No extensibility**: users of Symfony, Laravel, Slim, or PSR-7 stacks cannot integrate without wrapping the entire `view()` call.

The streaming design (ADR-0017) introduces `FileStream::serve()` which also needs to output HTTP responses. A single, clean abstraction is needed for both.

## Decision

Introduce a `ResponseEmitter` interface that abstracts HTTP output. The `serve()` method accepts an optional emitter; if none is provided, a default raw-output emitter is used.

### ResponseEmitter interface

```php
<?php
namespace WebFiori\File;

interface ResponseEmitter {
    public function setHeader(string $name, string $value): void;
    public function setStatusCode(int $code): void;
    public function sendBody(\Generator $chunks): void;
}
```

### Built-in implementations

```php
<?php
namespace WebFiori\File;

class DefaultEmitter implements ResponseEmitter {
    public function setHeader(string $name, string $value): void {
        header("$name: $value");
    }
    public function setStatusCode(int $code): void {
        http_response_code($code);
    }
    public function sendBody(\Generator $chunks): void {
        foreach ($chunks as $chunk) {
            if (connection_aborted()) break;
            echo $chunk;
            flush();
        }
    }
}
```

```php
<?php
namespace WebFiori\File;

use WebFiori\Http\Response;

class WebFioriEmitter implements ResponseEmitter {
    private Response $response;

    public function __construct(?Response $response = null) {
        $this->response = $response ?? new Response();
    }
    public function setHeader(string $name, string $value): void {
        $this->response->addHeader($name, $value);
    }
    public function setStatusCode(int $code): void {
        $this->response->setCode($code);
    }
    public function sendBody(\Generator $chunks): void {
        foreach ($chunks as $chunk) {
            $this->response->write($chunk);
        }
    }
    public function getResponse(): Response {
        return $this->response;
    }
}
```

### Usage in FileStream::serve()

```php
public function serve(bool $asAttachment = false, ?ResponseEmitter $emitter = null): void {
    $emitter = $emitter ?? new DefaultEmitter();
    // ... set headers via $emitter->setHeader() ...
    // ... set status via $emitter->setStatusCode() ...
    $emitter->sendBody($this->readChunks());
}
```

### Backward compatibility for File::view()

The existing `view()` method stays with its current signature. Internally it delegates to an emitter:

```php
public function view(bool $asAttachment = false, bool $terminate = true): void {
    // ... existing header logic refactored to use emitter internally ...
    if ($terminate && http_response_code() !== false) {
        die();
    }
}
```

Users who want the new emitter pattern use `$file->stream()->serve(emitter: ...)` instead.

## Alternatives Considered

- **Accept a callable instead of interface** — rejected because callables provide no type safety, no IDE support, and can't carry state (like a Response object to inspect after serving).
- **Adopt PSR-7 ResponseInterface directly** — rejected because it would require `psr/http-message` as a dependency. PSR-7 responses are also immutable, making streaming awkward. Users who want PSR-7 can write a thin `ResponseEmitter` adapter.
- **Remove HTTP serving entirely** — rejected because serving files with proper headers (Content-Type, Content-Disposition, byte ranges) is a core use case. Removing it would force every user to reimplement the same logic.
- **Keep `class_exists()` detection** — rejected because implicit behavior based on installed packages violates the principle of least surprise.

## Consequences

### Benefits

- `File.php` no longer imports framework classes — becomes truly standalone.
- HTTP serving behavior is explicit: caller chooses their emitter.
- Fully testable: inject a mock emitter, assert headers and body without touching real HTTP output.
- Works with any framework: PSR-7, Symfony, Laravel — just implement `ResponseEmitter`.
- `WebFioriEmitter` preserves existing behavior for WebFiori framework users.

### Trade-offs

- Users who relied on the automatic `Response` detection must now pass `new WebFioriEmitter()` explicitly when using `FileStream::serve()`. (Existing `File::view()` still auto-detects for BC.)
- One more interface and two more classes to maintain.

### Related

- ADR-0016: FileInterface (HTTP serving excluded from interface — this ADR explains where it lives instead)
- ADR-0017: Streaming (serve() uses generators from FileStream)
- ADR-0019: Framework decoupling (this ADR is the mechanism for that decoupling)
