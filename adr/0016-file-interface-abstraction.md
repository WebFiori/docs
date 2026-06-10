# ADR-0016: Introduce FileInterface Abstraction

**Date:** 2026-06-02
**Status:** Accepted

## Context

The `webfiori/file` library's `File` class is a concrete class that consumers must depend on directly. This creates several problems:

- **Tight coupling** — any code that uses `File` is locked to the local filesystem implementation. There is no way to swap in a different backend (S3, in-memory, FTP) without changing consumer code.
- **Untestable consumers** — services that accept or return `File` objects cannot be unit-tested without touching the real filesystem. Mocking a concrete class with private state is fragile.
- **God class** — `File` mixes file metadata, I/O operations, Base64 encoding, JSON serialization, and HTTP response serving. Without an interface defining the core contract, there is no clear boundary between what is essential file behavior and what is auxiliary.
- **No polymorphism** — `UploadedFile` extends `File`, but there is no shared contract that other file-like objects could implement.

The library needs a clear, minimal interface that defines what a "file" is in terms of behavior, separate from how it is implemented.

## Decision

Introduce a `FileInterface` in `WebFiori\File` that captures core file operations:

```php
<?php
namespace WebFiori\File;

interface FileInterface {
    public function getName(): string;
    public function setName(string $name): void;
    public function getDir(): string;
    public function setDir(string $dir): bool;
    public function getAbsolutePath(): string;
    public function getExtension(): string;
    public function getMIME(): string;
    public function getSize(): ?int;
    public function isExist(): bool;
    public function getRawData(bool $encode = false): string;
    public function setRawData(string $raw, bool $decode = false, bool $strict = false): void;
    public function append(string|array $data): void;
    public function read(int $from = -1, int $to = -1): void;
    public function write(bool $append = true, bool $createIfNotExist = false): void;
    public function create(bool $createDirIfNotExist = false): void;
    public function remove(): bool;
}
```

The `File` class implements this interface. `UploadedFile` inherits the implementation through `File`.

### What is excluded from the interface

- `view()` / `getResponse()` — HTTP serving is not a file concern; it belongs in a separate service.
- `toJSON()` / `__toString()` — serialization is orthogonal to file I/O.
- `writeEncoded()` / `readDecoded()` — Base64 encoding is a convenience layer, not core behavior.
- `toBytesArray()` / `toHexArray()` — data format conversion utilities.
- `getChunks()` — chunking strategy is implementation detail.
- `getLastModified()` — not all file backends support modification timestamps.
- `setId()` / `getID()` — database concerns, not file behavior.

These methods remain on `File` but are not part of the contract.

## Implementation Steps

| Step | Description | Issue |
|------|-------------|-------|
| 1 | Define `FileInterface` with method signatures | [webfiori/file#66](https://github.com/WebFiori/file/issues/66) |
| 2 | `File implements FileInterface` | [webfiori/file#67](https://github.com/WebFiori/file/issues/67) |
| 3 | Verify `UploadedFile` satisfies interface, add assertion test | [webfiori/file#68](https://github.com/WebFiori/file/issues/68) |
| 4 | Add `FileInterface` type-hints to `FileUploader` public API | [webfiori/file#69](https://github.com/WebFiori/file/issues/69) |
| 5 | Document interface usage and mocking examples | [webfiori/file#70](https://github.com/WebFiori/file/issues/70) |

## Alternatives Considered

- **Abstract base class instead of interface** — PHP only allows single inheritance. An abstract class would prevent consumers from extending their own base classes. Interfaces allow composition and multiple implementations.
- **Adopt Flysystem's adapter pattern** — Flysystem solves a different problem (filesystem abstraction with multiple backends). This library is focused on single-file operations with encoding, chunking, and upload handling. Adopting Flysystem would mean replacing the library entirely rather than improving it.
- **Do nothing** — leaves consumers coupled to a concrete class, making testing and future backend swaps impossible.

## Consequences

### Benefits

- Consumers type-hint `FileInterface` instead of `File`, enabling dependency injection and mocking.
- Clear contract separates "what a file does" from "how this file does it."
- Opens the door for future implementations (cloud storage, in-memory files for testing).
- No breaking changes — purely additive.

### Trade-offs

- Adds one more file to maintain.
- Developers must decide whether to type-hint `FileInterface` or `File` — the guidance is: use `FileInterface` when you only need I/O operations, use `File` when you need encoding/serialization/HTTP features.
- The interface locks method signatures — changing them later requires a major version bump.
