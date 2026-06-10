# ADR-0021: File Locking and Atomic Write Strategy

**Date:** 2026-06-02
**Status:** Proposed

## Context

The `File::write()` method opens a file and writes data without any concurrency protection:

```php
$resource = self::createResource('ab', $fPath);
fwrite($resource, $this->getRawData($encode));
fclose($resource);
```

In multi-process environments (web servers with multiple workers, cron jobs, queue workers), concurrent writes to the same file can interleave bytes and produce corrupted data. Additionally, if the process crashes mid-write (OOM, signal, power loss), the file is left in a partial/corrupt state with no recovery path.

Two separate problems require two solutions:
1. **Concurrency**: multiple writers must not interleave.
2. **Atomicity**: a write either completes fully or not at all.

## Decision

### File locking with `flock()`

Add advisory file locking to all write operations. This applies to both `File::write()` and `FileStream::writeFromStream()`.

```php
// File::write() — adds locking
public function write(bool $append = true, bool $createIfNotExist = false, bool $lock = true): void {
    $pathV = $this->checkNameAndPath();
    if ($createIfNotExist) {
        $this->create(true);
    }
    
    $mode = $append ? 'ab' : 'cb'; // 'cb' opens for writing without truncating
    $resource = fopen($pathV, $mode);
    
    if ($lock && !flock($resource, LOCK_EX)) {
        fclose($resource);
        throw new FileException("Unable to acquire exclusive lock on '$pathV'.");
    }
    
    if (!$append) {
        ftruncate($resource, 0);
        rewind($resource);
    }
    
    fwrite($resource, $this->getRawData());
    fflush($resource);
    
    if ($lock) {
        flock($resource, LOCK_UN);
    }
    fclose($resource);
}
```

```php
// FileStream::writeFromStream() — always supports locking
public function writeFromStream(iterable $source, bool $append = true, bool $lock = true): void {
    $mode = $append ? 'ab' : 'cb';
    $handle = fopen($this->path, $mode);
    
    try {
        if ($lock && !flock($handle, LOCK_EX)) {
            throw new FileException("Unable to acquire exclusive lock on '{$this->path}'.");
        }
        
        if (!$append) {
            ftruncate($handle, 0);
            rewind($handle);
        }
        
        foreach ($source as $chunk) {
            fwrite($handle, $chunk);
        }
        fflush($handle);
    } finally {
        if ($lock) {
            flock($handle, LOCK_UN);
        }
        fclose($handle);
    }
}
```

### Atomic writes via temp + rename

For non-append overwrites where crash safety matters, use the write-to-temp-then-rename pattern. This is exposed as a separate method since it has different semantics (creates a new inode):

```php
// FileStream
public function writeAtomic(iterable $source): void {
    $tempPath = $this->path . '.tmp.' . getmypid();
    $handle = fopen($tempPath, 'wb');
    
    if (!is_resource($handle)) {
        throw new FileException("Unable to create temp file at '$tempPath'.");
    }
    
    try {
        foreach ($source as $chunk) {
            fwrite($handle, $chunk);
        }
        fflush($handle);
        fclose($handle);
        $handle = null;
        
        if (!rename($tempPath, $this->path)) {
            throw new FileException("Atomic rename from '$tempPath' to '{$this->path}' failed.");
        }
    } finally {
        if (is_resource($handle)) {
            fclose($handle);
        }
        if (file_exists($tempPath)) {
            unlink($tempPath);
        }
    }
}
```

### Why `flock()` (advisory locking)

- Works on all platforms PHP supports (Linux, macOS, Windows).
- Cooperative: all writers using this library respect the lock. External tools (editors, other scripts) may not — but this is the standard trade-off of advisory locking.
- No external dependencies (no Redis, no database, no file-based mutexes).
- NFS caveat: `flock()` may not work reliably on NFS mounts. This is documented but not solved — solving it would require a distributed lock, which is out of scope.

### Default behavior

- `File::write()`: `$lock = true` by default. Backward compatible because adding a default parameter doesn't change the call signature.
- `FileStream::writeFromStream()`: `$lock = true` by default.
- `FileStream::writeAtomic()`: no lock needed — rename is atomic on POSIX filesystems.

## Alternatives Considered

- **No locking by default (opt-in)** — rejected because silent data corruption is worse than a small performance cost. Advisory locking overhead is negligible for typical file sizes.
- **Mandatory locking (OS-level)** — not available cross-platform in PHP. Linux deprecated mandatory locking in kernel 5.15.
- **External lock files (`.lock` sidecar)** — more complex, requires cleanup on crash, solves the same problem `flock()` solves natively.
- **Always use atomic writes** — rejected because atomic writes don't support append mode, and they create new inodes (breaking hardlinks and some monitoring tools).
- **Use `file_put_contents()` with `LOCK_EX` flag** — only works for one-shot writes. Doesn't support streaming or append with lock.

## Consequences

### Benefits

- Concurrent writes no longer produce corrupted files.
- `writeAtomic()` ensures crash-safe full-file overwrites.
- Default locking means users get safety without thinking about it.
- `$lock = false` available for performance-sensitive cases where the caller manages concurrency externally.

### Trade-offs

- Advisory locking only protects cooperating processes. Scripts that don't use `flock()` can still corrupt files.
- `writeAtomic()` requires the destination directory to be writable (for the temp file) and uses extra disk space momentarily.
- `rename()` is only atomic on the same filesystem. Cross-device atomic write falls back to copy + unlink (documented).
- NFS environments may need external locking — documented as a known limitation.

### Related

- ADR-0017: Streaming (FileStream::writeFromStream uses locking)
- ADR-0020: Backward compatibility ($lock defaults to true, new parameter with default — non-breaking)
