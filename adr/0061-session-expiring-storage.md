# ADR-0061: Session Expiry as Storage Data via `ExpiringSessionStorage`

**Date:** 2026-11-16
**Status:** Accepted

## Context

Since the per-key session system (ADR-0047, framework 3.1.0), the lifetime of
a session lives in the reserved `_meta` key (`startedAt`, `lifeTime`, `isRef`).
Garbage collection never reads `_meta`. It removes sessions with one global
idle threshold:

- `SessionManager::getGCTime()` returns "now minus 4 hours" (twice
  `Session::DEFAULT_SESSION_DURATION`), or a value from `SESSION_GC`.
- `SessionStorage::gc(string $olderThan, int $maxCount)` deletes every session
  last used at or before that time. `SessionDB` compares it to the
  `sessions.last_used` column, `DefaultSessionStorage` to the file's mtime.

This caused a group of bugs, all reproduced on `dev` against every built-in
driver (default file, in-memory, cache, SQLite, MySQL 8, SQL Server 2022):

| Issue | Problem |
|-------|---------|
| [#432](https://github.com/WebFiori/framework/issues/432) | A session with a lifetime longer than 4 hours (for example a 7 day "remember me" session) is deleted after 4 idle hours. |
| [#433](https://github.com/WebFiori/framework/issues/433) | Resuming and reading a session does not update its last-use time, so an active session looks idle to GC. |
| [#431](https://github.com/WebFiori/framework/issues/431) | `setDuration()` and `setIsRefresh()` only change memory, so the stored lifetime stays the one from creation. |
| [#434](https://github.com/WebFiori/framework/issues/434) | `SESSION_GC` is read as an absolute Unix timestamp, so no static value is useful. |
| [#435](https://github.com/WebFiori/framework/issues/435) | A missing `lifeTime` in `_meta` falls back to 2 minutes instead of 120. |

#431, #434 and #435 have local fixes. #432 and #433 have one root cause: the
expiry of a session is a fact only `Session` knows, but GC has to decide
without it. Each storage then guesses from a last-use timestamp and a global
threshold.

**Constraints:**

- `SessionStorage` is a public interface since 3.1.0, and applications may
  implement it. Adding a method to it breaks those implementations, which
  is not acceptable in a 3.x minor release.
- Custom storages written for the pre-3.1 contract are wrapped in
  `LegacySessionStorageAdapter` and must keep working.
- Existing databases have no column for expiry and must keep working before
  and after the schema migration.

## Decision

Expiry becomes data that the storage keeps and enforces with its native
mechanism. `Session` is the only component that computes it.

### 1. New optional interface

```php
namespace WebFiori\Framework\Session;

interface ExpiringSessionStorage extends SessionStorage {
    public function getExpiry(string $sessionId): ?int;
    public function setExpiry(string $sessionId, ?int $expiresAt): void;
}
```

`$expiresAt` is a Unix timestamp. `null` means "no recorded expiry". The
idle threshold passed to `gc()` then applies, as before.

Contract for `gc()` on an `ExpiringSessionStorage`:

- A session with a recorded expiry is removed once the expiry has passed,
  whatever the idle threshold.
- A session without one is removed by the idle threshold.

### 2. `Session` computes and pushes the expiry

- Expiry is `startedAt + duration` for a session without refresh, and
  `now + duration` for a session with refresh. A non-persistent session
  (duration 0) has no expiry (`null`).
- It is written whenever `_meta` is written: on creation, on `close()`, and
  now also from `setDuration()` and `setIsRefresh()` on a running session
  (the fix for #431).
- On resume, the expiry is written again if none is recorded (sessions
  created before the upgrade), or if it drifted from the computed value by
  more than 60 seconds. Drift only happens for sessions with refresh. This
  replaces "update last-use time on resume" as the fix for #433. A busy
  refresh session costs at most one expiry write per minute, not one per
  request.
- `Session` checks `instanceof ExpiringSessionStorage`. Other storages get no
  calls and behave as in 3.1.

### 3. Each built-in driver uses its native mechanism

| Driver | Where the expiry lives | How `gc()` uses it |
|--------|------------------------|--------------------|
| `DatabaseSessionStorage` | Nullable `sessions.expires_at` column with index `idx_sessions_expires_at` | Removes rows with `expires_at <= now`, plus rows with `expires_at IS NULL` and `last_used <= $olderThan`. |
| `DefaultSessionStorage` | Sidecar file `<id>.exp` next to the session file (mtime changes on every write, so it cannot hold the expiry) | Compares the sidecar to now. Without a sidecar it compares the session file mtime to `$olderThan`. Removes orphan sidecars. |
| `CacheSessionStorage` | An `<id>:_expires` cache entry. All entries of the session get a TTL equal to the remaining lifetime, replacing the fixed 7200 s default | The cache backend expires entries itself, and `gc()` stays `purgeExpired()`. |
| `InMemorySessionStorage` | A static map | Removes sessions whose expiry passed. It still has no idle tracking. |

### 4. Schema migration and degraded mode

`SessionSchemaMigration::run()` adds `sessions.expires_at` (`DATETIME NULL`)
and its index. It is idempotent and works on MySQL, SQL Server and SQLite.
`createTables()` creates the column for new installs.

Until the migration runs, `SessionDB` detects the missing column:
`getExpiry()` returns `null`, `setExpiry()` does nothing, and `gc()` falls back
to the idle threshold. Sessions behave exactly as in 3.1.

### 5. `SESSION_GC` is a duration (#434)

`SESSION_GC` (env var or constant) is now an idle period in seconds. The
threshold is `time() - SESSION_GC`. With expiry recorded, it only affects
sessions without one. A static timestamp could never express a moving
threshold, so we accept the change of meaning without a deprecation period.

## Alternatives Considered

**A. Add `setExpiry()`/`getExpiry()` to `SessionStorage` now.** This is the
correct end state (see Future Direction). It breaks every custom
`SessionStorage` implementation in a minor release. Rejected for 3.x.

**B. GC reads `_meta` for each candidate session (no schema change).** The
global threshold would only pre-select candidates. GC would read `_meta`,
rebuild the expiry and decide. Rejected:

- The expiry rule would exist twice, in `Session` and in GC code, and each
  driver would need a copy.
- GC would cost one read per candidate.
- The arbitrary global threshold would stay as a filter.
- #433 would still need a throttled `_meta` rewrite to keep last-use accurate.

**C. Only add an `expires_at` column to the database driver.** It fixes the
database case but leaves the file, cache and in-memory drivers with the old
behavior, and puts the expiry rule inside `SessionDB`. Rejected in favor of
an interface every driver implements.

**D. Raise the default GC threshold (for example to 7 days).** It hides #432
for most lifetimes but keeps expired short sessions in storage for days,
and still ignores per-session lifetime. Rejected.

## Consequences

- Sessions live exactly as long as their own lifetime. Long-lived sessions are
  no longer cut at 4 idle hours, so storage holds more rows than before. This
  is the intended behavior.
- `Session` is the single source of the expiry rule. Drivers only store and
  enforce a timestamp.
- Database GC is an indexed range query instead of a scan by `last_used`.
- The cache driver's TTL follows the session lifetime instead of a fixed
  7200 seconds.
- Deployments must run `SessionSchemaMigration::run()` to get expiry-based GC
  on database storage. Without it, behavior is unchanged from 3.1.
- Custom storages that do not implement `ExpiringSessionStorage` keep the old
  idle-threshold behavior, including #432 and #433. They can opt in by
  implementing the two methods.
- Anyone who set `SESSION_GC` to an absolute timestamp now gets an idle period
  of that many seconds (about 57 years for a current timestamp), which keeps
  sessions without a recorded expiry forever. We consider such usage unlikely.
- Each resume of a session that has refresh does at most one extra storage
  write per minute.

## Future Direction

`ExpiringSessionStorage` is a transition step forced by backward
compatibility. The architecturally correct design is for expiry to be part of
the core storage contract. In the next major version (4.0):

1. Move `getExpiry()` and `setExpiry()` into `SessionStorage` and remove
   `ExpiringSessionStorage`. It can be kept for one release as an empty
   deprecated alias that extends `SessionStorage`.
2. Change `gc()` to `gc(int $maxCount = 0): void`, meaning "remove expired
   sessions". The `$olderThan` idle threshold, `SessionManager::getGCTime()` and
   `SESSION_GC` would then only exist for `LegacySessionStorageAdapter`, or be
   removed.
3. Remove the `instanceof ExpiringSessionStorage` checks from `Session` and the
   degraded mode in `SessionDB` (require the migrated schema).
4. Consider making `expires_at` `NOT NULL` once all rows have been written by
   3.2+ code.
