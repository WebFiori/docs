# ADR-0017: Streaming File I/O with FileStream

**Date:** 2026-06-02
**Status:** Accepted

## Context

The `File` class loads entire file contents into a private `$rawData` string property. Every operation — `read()`, `write()`, `view()`, `getChunks()`, `toBytesArray()` — works on this in-memory blob. This makes the library unusable for files that approach or exceed PHP's memory limit.

Real-world impact:
- Serving a 500MB video: 500MB+ of PHP memory consumed before a single byte reaches the client.
- Processing a 1GB log file: fatal `Allowed memory size exhausted` error.
- `getChunks()` appears to support incremental processing but actually requires the full data loaded first, then splits it into array pieces — using *more* memory.

The library needs a streaming mode that processes files in constant memory regardless of file size, while preserving the existing one-shot API for small files where simplicity matters.

## Decision

Introduce a `StreamableInterface` and a `FileStream` class that operates on file handles with fixed-size buffers. This is a **separate class**, not a modification of `File`.

### StreamableInterface

```php
<?php
namespace WebFiori\File;

interface StreamableInterface {
    public function readChunks(int $bufferSize = 8192): \Generator;
    public function readLines(): \Generator;
    public function readRange(int $from, int $to, int $bufferSize = 8192): \Generator;
    public function writeFromStream(iterable $source, bool $append = true, bool $lock = true): void;
    public function serve(bool $asAttachment = false, ?ResponseEmitter $emitter = null): void;
    public function getSize(): int;
    public function getMIME(): string;
    public function getName(): string;
}
```

### FileStream class

```php
<?php
namespace WebFiori\File;

class FileStream implements StreamableInterface {
    private string $path;
    private int $bufferSize;

    public function __construct(string $path, int $bufferSize = 8192);
    public function readChunks(int $bufferSize = 8192): \Generator;
    public function readLines(): \Generator;
    public function readRange(int $from, int $to, int $bufferSize = 8192): \Generator;
    public function writeFromStream(iterable $source, bool $append = true, bool $lock = true): void;
    public function serve(bool $asAttachment = false, ?ResponseEmitter $emitter = null): void;
    public function getSize(): int;
    public function getMIME(): string;
    public function getName(): string;
    public function getBufferSize(): int;
    public function setBufferSize(int $size): void;
}
```

### Bridge from File

```php
// Added to File class
public function stream(int $bufferSize = 8192): FileStream {
    return new FileStream($this->getAbsolutePath(), $bufferSize);
}
```

### Why generators

- Generators yield one chunk at a time — memory is constant.
- Callers use `foreach`, which is idiomatic PHP.
- Generators are lazy — if the caller breaks early, the `finally` block closes the file handle.
- Generators compose: `writeFromStream($stream->readChunks())` pipes data with no intermediate buffer.

### Why a separate class (not extending File)

- `File` stores state in `$rawData`. Streaming is stateless — it operates on handles. Mixing both in one class creates conflicting invariants.
- `File` implements `JsonI` and exposes `toJSON()`, `getChunks()`, `toBytesArray()` — none of which make sense for a stream.
- Separate class means no risk of breaking existing `File` behavior.
- Users choose explicitly: `$file->read()` (one-shot) vs `$file->stream()->readChunks()` (streaming).

## Alternatives Considered

- **Add streaming methods directly to `File`** — rejected because it would bloat an already large class (36KB) and create ambiguity about whether data lives in `$rawData` or is being streamed.
- **Use PSR-7 `StreamInterface`** — rejected because it adds a dependency on `psr/http-message` and its API is designed for HTTP message bodies, not general file I/O (no `readLines()`, no `readRange()`, no generators).
- **Return arrays instead of generators** — rejected because arrays require all chunks in memory simultaneously, defeating the purpose.
- **Callback-based chunking (`readChunks(callable $callback)`)** — rejected because generators are more composable and testable.

## Consequences

### Benefits

- Any file can be processed in constant ~8KB memory regardless of size.
- Streaming compose naturally: read from one file, write to another, transform in between.
- `serve()` sends data to the client as it reads from disk — first byte arrives immediately.
- Existing `File` API is completely unchanged — zero migration cost.

### Trade-offs

- Two ways to do things: `$file->read()` + `$file->getRawData()` vs `$file->stream()->readChunks()`. Guidance: use one-shot for files < 10MB, streaming for anything larger or when memory matters.
- `FileStream` cannot provide `toJSON()` or `toBytesArray()` — these inherently require full data in memory.
- Generators cannot be rewound. To re-read, create a new generator.

### Related

- ADR-0016: FileInterface abstraction (FileStream does not implement FileInterface — different concern)
- ADR-0018: ResponseEmitter strategy (used by `FileStream::serve()`)
- ADR-0021: File locking (used by `FileStream::writeFromStream()`)
