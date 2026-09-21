# ADR-0047: Real-Time Session System with Conflict Resolution

**Date:** 2026-09-20
**Status:** Proposed

> **Amended 2026-09-21:** The original ADR is updated based on the concrete
> production scenario and design decisions made during implementation planning:
>
> **Real motivating scenario — IIS/FastCGI multi-worker:** The race condition
> is not hypothetical. Under IIS with FastCGI (or PHP-FPM with >1 worker),
> two worker processes can serve requests for the same session concurrently.
> Process 1 calls `set('notification', 'new-message')` mid-flight; Process 2
> (already running, holding a stale in-memory snapshot) never sees it. When
> Process 2 saves its whole-session blob at request end, it clobbers Process
> 1's key. This is reproducible on any multi-process PHP host.
>
> **Design deviation 1 — Non-throwing default:** The original ADR defaulted
> `set()` to `ConflictStrategy::REJECT` (throw `SessionConflictException`).
> Amended: the default is `ConflictStrategy::LAST_WRITE_WINS` (overwrite, no
> exception). `REJECT` and `RETRY_WITH_CALLBACK` are explicit opt-ins.
> Rationale: existing `$session->set(...)` call sites must continue to work
> without try/catch; the visibility fix (real-time reads) is the primary
> requirement for the IIS scenario.
>
> **Design deviation 2 — Configurable read strategy:** Read strategy is
> configurable at session level: `ReadStrategy::REALTIME` (default — every
> `get()` hits storage), `SNAPSHOT_WITH_MISS` (snapshot at start, live read
> on cache miss), `MANUAL_SYNC` (snapshot, developer calls `refresh()`
> explicitly). Default is `REALTIME` — correct for IIS/FastCGI out of the box.
>
> **Configuration point — `StartSessionMiddleware` constructor:** Strategies
> are passed as constructor arguments at route-registration time, not as
> static setters. `new StartSessionMiddleware()` (no args) gives safe
> defaults. Different routes may use different strategies.
>
> **Backward compatibility — `LegacySessionStorageAdapter`:** A shim wraps
> the old `read()/save()` interface in the new per-key contract. It provides
> NO conflict detection (concurrent writes may still lose data — last-writer-
> wins on the whole session blob). Documented clearly. Migrate to a native
> `SessionStorage` implementation to gain real-time reads and per-key
> conflict resolution.
>
> **Test-first:** Failing tests (`SessionVisibilityTest`,
> `SessionLostWriteTest`) are written before any implementation and confirmed
> failing against the current snapshot-isolation code.

## Context

The current WebFiori session implementation uses a **snapshot isolation** model:

1. Session data is loaded entirely into memory at request start
2. Changes are made to the in-memory copy during the request
3. The entire session is written back to storage at request end

This causes a **race condition** when multiple concurrent requests modify the same session:

```
Request A                          Request B
─────────                          ─────────
start() → read {user: 1}           
                                   start() → read {user: 1}
set('foo', 'bar')                  
                                   set('theme', 'dark')
end() → save {user:1, foo:'bar'}   
                                   end() → save {user:1, theme:'dark'}
                                   ← 'foo' is LOST
```

This affects all storage backends (database, file, cache) because the problem is in the session lifecycle design, not the storage layer.

**Technical constraints:**
- Must work with all storage backends (database, file, Redis, etc.)
- Must not block long-running requests (e.g., SSE streaming)
- Must maintain backward compatibility where possible
- Must be storage-agnostic in design

## Decision

Redesign the session system to use **real-time reads**, **immediate writes**, and **conflict resolution strategies**.

### Core Principles

1. **Real-time reads**: Every `get()` reads the current value from storage
2. **Immediate writes**: Every `set()` writes to storage immediately, not batched
3. **Conflict detection**: Use version tracking to detect concurrent modifications
4. **Conflict resolution**: Apply configurable strategy when conflict is detected

### Conflict Resolution Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| `REJECT` (default) | Throw `SessionConflictException` | Caller handles conflict explicitly |
| `FORCE` | Overwrite regardless of conflicts | When caller has authoritative value |
| `RETRY_WITH_CALLBACK` | Re-read current value, call user callback to compute new value | Complex merge logic (counters, aggregations) |

### API Design

**SessionStorage interface:**

```php
interface SessionStorage {
    public function read(string $sessionId, string $key): ?array; // {value, version}
    public function readAll(string $sessionId): array;
    public function write(
        string $sessionId, 
        string $key, 
        mixed $value, 
        string|int|null $expectedVersion = null,
        ConflictStrategy $strategy = ConflictStrategy::REJECT
    ): string|int;
    public function remove(string $sessionId, string $key): void;
    public function destroy(string $sessionId): void;
    public function gc(string $olderThan, int $maxCount = 0): void;
}

enum ConflictStrategy {
    case REJECT;
    case FORCE;
    case RETRY_WITH_CALLBACK;
}
```

**Session class:**

```php
class Session {
    public function get(string $key, mixed $default = null): mixed;
    public function has(string $key): bool;
    public function set(
        string $key, 
        mixed $value, 
        ConflictStrategy $strategy = ConflictStrategy::REJECT,
        ?callable $conflictCallback = null
    ): bool;
    public function remove(string $key): bool;
    public function setMany(array $values, ConflictStrategy $strategy = ConflictStrategy::REJECT): bool;
}
```

### Storage Schema (Database)

```sql
CREATE TABLE sessions (
    session_id VARCHAR(128) PRIMARY KEY,
    created_at DATETIME NOT NULL,
    last_activity DATETIME NOT NULL
);

CREATE TABLE session_data (
    session_id VARCHAR(128) NOT NULL,
    key VARCHAR(255) NOT NULL,
    value TEXT NOT NULL,
    version INT NOT NULL DEFAULT 1,
    updated_at DATETIME NOT NULL,
    PRIMARY KEY (session_id, key),
    FOREIGN KEY (session_id) REFERENCES sessions(session_id) ON DELETE CASCADE
);
```

### Migration Path

1. Add version column to existing session_data table
2. Implement new SessionStorage interface alongside existing
3. Update Session class with backward compatibility flag
4. Deprecate old interface in next minor, remove in next major version

## Alternatives Considered

1. **Pessimistic locking (lock entire session on read)**
   - Rejected: Serializes all requests for same user, blocks long-running requests like SSE

2. **Delta tracking with merge on save**
   - Rejected: More complex implementation, still has race window between read and save

3. **Event sourcing (append-only changes)**
   - Rejected: Too complex for typical session use cases, storage grows unbounded

4. **Last-write-wins with timestamp**
   - Rejected: Still loses data, just more predictably

## Consequences

**Positive:**
- No more lost writes from concurrent requests
- Predictable behavior — developers know when data is persisted
- Explicit conflict handling — caller decides resolution strategy
- Storage agnostic — design works for database, file, Redis, etc.

**Negative:**
- Breaking change to SessionStorage interface
- More I/O operations (real-time reads/writes vs batched)
- Added complexity from conflict strategies

**Migration:**
- Existing applications need to handle `SessionConflictException` or use `FORCE` strategy
- Database schema requires migration to add version column
- Performance testing recommended for high-traffic applications
