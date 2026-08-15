# ADR-0034: AI: SQLite3 Class Over PDO for SqliteVectorStore

**Date:** 2026-08-16
**Status:** Accepted

## Context

`SqliteVectorStore` needs to persist vectors to a SQLite database. PHP provides
two built-in ways to access SQLite:

1. **`SQLite3` class** — Direct PHP extension binding to `libsqlite3`
2. **`PDO` with `pdo_sqlite` driver** — PDO's own binding to `libsqlite3`

Both are zero-dependency (no composer packages). Both talk directly to
`libsqlite3` — PDO is **not** a wrapper on top of `SQLite3`; they are
independent implementations:

```
┌─────────────┐     ┌─────────────┐
│  SQLite3    │     │    PDO      │
│   class     │     │  pdo_sqlite │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └───────┬───────────┘
               │
        ┌──────▼──────┐
        │  libsqlite3 │
        └─────────────┘
```

## Decision

Use the `SQLite3` class directly.

```php
$this->db = new SQLite3($databasePath);
$this->db->enableExceptions(true);

$stmt = $this->db->prepare('SELECT id, vector, metadata FROM vectors WHERE id = :id');
$stmt->bindValue(':id', $id, SQLITE3_TEXT);
$result = $stmt->execute();
$row = $result->fetchArray(SQLITE3_ASSOC);
```

Type binding uses explicit `SQLITE3_TEXT` and `SQLITE3_INTEGER` constants,
which avoids the PDO type-inference bug where integer values passed through
`execute([])` are treated as `null` by SQLite's JSON functions.

## Alternatives Considered

**PDO with `pdo_sqlite` driver:**

Rejected for two reasons:

1. **Extension availability** — `pdo_sqlite` is commonly enabled but is a
   separate extension from `sqlite3`. Minimal PHP installations (Docker images,
   shared hosts) sometimes omit it. The `sqlite3` extension is enabled by
   default in official PHP builds.

2. **Type inference bug with `execute()` arrays** — When filtering by integer
   metadata values using `json_extract()`, passing parameters via
   `$stmt->execute([$path, $intValue])` causes SQLite to treat the integer as
   `null`. The workaround requires `bindValue()` with explicit `PDO::PARAM_INT`,
   which removes the advantage of PDO's uniform parameter API.

   The `SQLite3` class requires explicit `SQLITE3_INTEGER` binding upfront,
   making the correct behavior the only option.

## Consequences

**Easier:**
- No risk of `pdo_sqlite` being missing on minimal PHP installations
- Explicit type constants (`SQLITE3_TEXT`, `SQLITE3_INTEGER`) make type handling
  unambiguous
- `__destruct()` with `$this->db->close()` makes connection lifecycle explicit

**Harder:**
- API is less familiar to developers who only know PDO
- Cannot reuse PDO connection pooling patterns (not relevant for this use case)
- If a developer wants to extend `SqliteVectorStore` for MySQL/PostgreSQL,
  they must implement `VectorStorageInterface` from scratch rather than
  swapping the DSN string (which is the correct approach anyway — vector search
  logic differs per backend)
