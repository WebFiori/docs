# ADR-0058: AI: SqlServerVectorStore — SQL Server-Backed VectorStorageInterface

**Date:** 2026-10-25
**Status:** Proposed

## Context

The library ships three `VectorStorageInterface` implementations —
`SqliteVectorStore`, `FileVectorStore`, and `InMemoryVectorStore` — but none
backed by a networked RDBMS. `SqliteVectorStore` stores vectors as JSON blobs
and runs brute-force cosine similarity in PHP (see ADR-0034 for why it uses the
`SQLite3` class over PDO).

For applications standardized on **Microsoft SQL Server**, this forces RAG and
agent-memory vectors (ADR-0045, ADR-0046) into per-host SQLite files under a
`RAG_PATH`. In multi-instance / multi-portal deployments this causes real
friction (per issue #175):

- **No single source of truth** — each host has its own `.db` files.
- **A hard writable-filesystem dependency** — on Windows/IIS this surfaces as
  `attempt to write a readonly database` when the app-pool identity lacks write
  access to `RAG_PATH`.
- **Vectors sit outside the SQL Server backup / HA / monitoring story** and need
  separate backup handling.
- **SQLite's single-writer model** is awkward under concurrent writes.

There is currently no supported way to keep embeddings in the same RDBMS the
rest of the application already uses.

ADR-0034 explicitly anticipated this situation:

> "If a developer wants to extend `SqliteVectorStore` for MySQL/PostgreSQL, they
> must implement `VectorStorageInterface` from scratch rather than swapping the
> DSN string ... (which is the correct approach anyway — vector search logic
> differs per backend)."

## Decision

Add `SqlServerVectorStore` implementing `VectorStorageInterface`, backed by the
`sqlsrv` PHP extension, so RAG and agent-memory vectors can be stored centrally
in SQL Server alongside the application's other data. It is a new
`LocalRagProvider` backend (ADR-0045) — not a `RagProvider` — and preserves the
public surface of `SqliteVectorStore` so it is a drop-in at the
`VectorStorageInterface` level.

Namespace: `WebFiori\Ai\Embedding\SqlServerVectorStore`.

### Design Decisions

**1. Driver: `sqlsrv` extension (not `pdo_sqlsrv`)**

`sqlsrv` and `pdo_sqlsrv` are independent Microsoft extensions over the same
ODBC driver (neither wraps the other) — the same relationship ADR-0034 notes
between the `SQLite3` class and `pdo_sqlite`. We use `sqlsrv` because it matches
WebFiori's own data-access layer (there is no PDO handle to reuse), it allows
explicit per-parameter type/direction binding (avoiding the integer-binding
foot-gun class ADR-0034 flagged for PDO), and it provides native stream handling
for large `NVARCHAR(MAX)` / `VARBINARY(MAX)` fields.

**2. Constructor mirrors `SqliteVectorStore`**

The store owns, opens, and closes its own connection — it does not accept a
pre-existing handle, matching `SqliteVectorStore(string $databasePath, array $options)`.

```php
public function __construct(string $serverName, array $options = [])
```

`$serverName` is the `sqlsrv_connect()` server string; `$options` carries both
the connection info and store tuning (see below). The connection is opened in
the constructor and closed in `__destruct()` via `sqlsrv_close()`.

**3. Options via a `SqlServerOption` constants class**

Option keys are defined by a `final class WebFiori\Ai\Embedding\SqlServerOption`
(not instantiable), mirroring the `ChatOption` convention — string constants with
per-key doc comments, for IDE discoverability and typo safety. A PHP `enum` is
deliberately not used, to stay consistent with `ChatOption`.

**4. Split options: owned outer keys, driver passthrough for connection info**

```php
new SqlServerVectorStore('tcp:myserver,1433', [
    SqlServerOption::CONNECTION_INFO => [      // passed verbatim to sqlsrv_connect()
        SqlServerOption::CONN_DATABASE => 'app_db',
        SqlServerOption::CONN_UID      => 'app_user',
        SqlServerOption::CONN_PWD      => '...',
        SqlServerOption::CONN_ENCRYPT  => true,
        // any other sqlsrv key may be passed as a raw string
    ],
    SqlServerOption::TABLE       => 'vectors', // store tuning (ours, validated)
    SqlServerOption::AUTO_CREATE => false,
]);
```

- Outer keys (`CONNECTION_INFO`, `TABLE`, `AUTO_CREATE`, …) are **ours** — read
  and validated by the store.
- `CONNECTION_INFO` is a nested array passed **verbatim** to `sqlsrv_connect()`.
  Its keys are Microsoft's, not ours. We provide a **convenience subset** of
  constants (`CONN_DATABASE`, `CONN_UID`, `CONN_PWD`, `CONN_ENCRYPT`, …) whose
  values equal the exact driver strings; any key not covered still passes through
  as a raw string. The subset is explicitly non-exhaustive — a missing constant
  does not mean the option is unsupported.

**5. Storage format: portable-first (`NVARCHAR(MAX)` JSON + PHP-side cosine)**

v1 stores vectors as `NVARCHAR(MAX)` JSON (parity with `SqliteVectorStore`) and
computes cosine similarity in PHP over candidate rows. The metadata `filter` is
pushed into the SQL `WHERE` clause so only candidate rows cross the wire, then
cosine is computed, sorted descending, and sliced to `topK` — identical control
flow to `SqliteVectorStore::query()`.

The native SQL Server 2025 `VECTOR` type + `VECTOR_DISTANCE()` server-side
scoring path is **deferred to a follow-up ADR**. The `SIMILARITY_MODE` /
`VECTOR_COLUMN_TYPE` options are therefore omitted from v1 (one mode, one column
type) and will be introduced when the native tier lands.

**6. Minimum version: SQL Server 2016**

Pushing the metadata filter into `WHERE` requires `JSON_VALUE()`, introduced in
SQL Server 2016 (and available on all current Azure SQL). 2016 is therefore a
hard minimum, not a preference — older versions would force all filtering into
PHP, defeating the "only candidate rows cross the wire" goal.

**7. Filter semantics: identical to `SqliteVectorStore`, parameterized**

The filter is a flat `array<string,mixed>` of equality pairs, AND-combined, over
top-level metadata keys, with bool coerced to int — exactly matching
`SqliteVectorStore`. The SQL Server equivalent of
`json_extract(metadata,'$.key')` is `JSON_VALUE(metadata,'$.key')`:

```sql
WHERE JSON_VALUE(metadata, '$.key1') = ? AND JSON_VALUE(metadata, '$.key2') = ?
```

One deliberate deviation: SQLite interpolates escaped values into the SQL string;
`SqlServerVectorStore` uses **parameterized `sqlsrv_query($conn, $sql, $params)`**
instead — safer (no injection surface) and leveraging the explicit-typing
advantage that motivated choosing `sqlsrv`. Because `JSON_VALUE` returns text,
numeric/bool filters mirror SQLite's coercion (bool → `0/1`, numbers compared as
their JSON text form).

**8. Upsert: UPDATE-then-INSERT in a transaction (not `MERGE`)**

SQL Server has no `INSERT OR REPLACE`. `store()` performs
`UPDATE ... WHERE id = ?`; if `@@ROWCOUNT = 0`, it `INSERT`s — wrapped in a
transaction. `MERGE` is avoided because of its well-documented concurrency
foot-guns (deadlocks/races without `HOLDLOCK`/`SERIALIZABLE` hints); since one
motivation for this store is SQLite's poor concurrency, reintroducing a
concurrency foot-gun would be self-defeating. `storeBatch()` runs all records in
a single transaction with rollback, mirroring `SqliteVectorStore`.

**9. Schema auto-creation: `AUTO_CREATE` option, default `false`**

Unlike `SqliteVectorStore` (which auto-creates unconditionally), the default is
`false`. The recommended deployment is a dedicated, least-privilege SQL Server
account that typically lacks DDL rights; an unconditional `CREATE TABLE` on
construction would fail exactly where this store is meant to help. The divergence
from SQLite's default is intentional and justified by the different trust model.

- `AUTO_CREATE => true` issues idempotent DDL (`IF OBJECT_ID(...) IS NULL`),
  for local/dev convenience.
- A public **static schema-DDL helper** returns the `CREATE TABLE` + index SQL so
  a DBA or a privileged setup command can provision the table out-of-band, then
  run the app under the restricted account with `AUTO_CREATE => false`.

Schema:

```sql
CREATE TABLE vectors (
  id         NVARCHAR(255) NOT NULL PRIMARY KEY,
  vector     NVARCHAR(MAX) NOT NULL,            -- JSON array
  metadata   NVARCHAR(MAX) NOT NULL DEFAULT '{}',
  created_at BIGINT        NOT NULL
);
CREATE INDEX idx_vectors_created_at ON vectors(created_at);
```

**10. Parity surface**

Implements `store`, `storeBatch`, `query(vector, topK, filter)`, `get`,
`delete(): bool`, `count`, `clear`. SQLite-specific methods `vacuum()` and
`getDatabasePath()` are intentionally not carried over (no equivalent); errors
map to the library's exceptions (`InvalidArgumentException`, `RuntimeException`)
per ADR-0030, translating `sqlsrv_errors()`.

### Testing (Docker)

Integration tests run against SQL Server in Docker
(`mcr.microsoft.com/mssql/server:2022-latest` — ≥2016, exercises `JSON_VALUE`).
A DB-only `docker-compose.yml` lives under `tests/docker/`; PHP-side setup
(`msodbcsql18` + `pecl install sqlsrv`) is documented. Tests **skip when no SQL
Server is reachable** (via an env var such as `MSSQL_SERVER`), so the existing
php81–php85 CI matrix stays green without driver changes. Wiring a dedicated CI
job is out of scope for v1.

### Example

A runnable example in `examples/32-sqlserver-vector-store/` uses **synthetic
vectors** (no AI API key required — only the Docker DB), demonstrating store
CRUD, metadata-filtered `query()`, and top-K scoring, plus a README note on
plugging the store into `LocalRagProvider` (ADR-0045). A live example using
Google/Gemini embeddings (via a service-account key) is a planned follow-up.

> Validation note: `sqlsrv` is a non-default extension requiring Microsoft's ODBC
> driver. The example and integration tests can only be executed where `sqlsrv`
> is installed; verification status is reported honestly rather than assumed.

## Alternatives Considered

**`pdo_sqlsrv` driver.** Rejected: WebFiori's data-access layer holds no PDO
handle, so the "reuse the app's connection" benefit does not apply; `sqlsrv`
gives explicit parameter typing and better LOB streaming.

**`MERGE` for upsert.** Rejected for v1: documented concurrency/locking
foot-guns conflict with the store's concurrency motivation; UPDATE-then-INSERT in
a transaction is simpler and robust.

**Native `VECTOR` / `VECTOR_DISTANCE()` in v1.** Deferred: requires SQL Server
2025 / recent Azure SQL to even test, and doubles the query/storage code paths.
The portable PHP-side path solves issue #175's stated pain on all supported
versions today; native scoring is a clean additive follow-up.

**`AUTO_CREATE` default `true` (SQLite parity).** Rejected: the recommended
least-privilege account lacks DDL rights, so `true` would fail on first
construction in the recommended setup. `false` + an explicit provisioning helper
matches how SQL Server schema is actually managed.

**Keep SQLite and fix only filesystem permissions.** Rejected: solves the write
errors but not centralization, HA/backup, or the multi-host single-source-of-
truth need.

**Dedicated vector database (pgvector, managed service).** Better for large-scale
ANN, but adds infrastructure for teams otherwise all-in on SQL Server.

## Consequences

**Easier:**

- Embeddings live in the same RDBMS as the rest of the app — one source of
  truth, covered by existing backup / HA / monitoring.
- Removes the writable-`RAG_PATH` dependency that breaks on Windows/IIS.
- Drop-in at the `VectorStorageInterface` level — swapping from
  `SqliteVectorStore` needs no caller changes; works under `LocalRagProvider`,
  `AgentMemory`, and `RetrievalTool` unchanged.
- Parameterized queries remove the SQL-injection surface present in the SQLite
  store's string interpolation.

**Harder / trade-offs:**

- Requires the `sqlsrv` extension + Microsoft ODBC driver — a non-default
  dependency, and the main thing likely to complicate CI and local setup.
- Requires SQL Server 2016+ (for `JSON_VALUE`).
- Default `AUTO_CREATE => false` diverges from `SqliteVectorStore`, so a quick
  local trial must opt in or run the schema helper (documented).
- PHP-side cosine does not scale to very large corpora; native server-side
  scoring awaits the follow-up `VECTOR` ADR.
- `vacuum()` / `getDatabasePath()` are not available (intentional non-parity).

**Backward compatibility:** purely additive. A new `VectorStorageInterface`
implementation alongside the existing stores; existing consumers are unaffected.
