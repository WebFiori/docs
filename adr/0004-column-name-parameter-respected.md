# ADR-0004: Column Attribute Name Parameter Must Be Respected

**Date:** 2026-05-31
**Status:** Accepted

## Context

The `#[Column(name: 'created_at')]` attribute has a `name` parameter intended to set the actual database column name. However, `TableFactory::create()` ignores this value and always uses the array key derived from `propertyToKey()` (which converts camelCase property names to kebab-case).

This results in:
- `#[Column(name: 'created_at')]` on property `$createdAt` → DB column `created-at` (not `created_at`)
- Hyphenated column names require bracket-escaping on MSSQL
- Raw SQL queries must use `[created-at]` instead of the expected `created_at`
- No database engine defaults to hyphenated column names

The `ColOption::NAME` value is stored in the options array but never read during column creation — it's dead data.

## Decision

In `TableFactory::create()`, use `$options[ColOption::NAME]` as the column name when explicitly set. Fall back to the array key (from `propertyToKey()`) when no explicit name is provided.

```php
foreach ($cols as $key => $options) {
    $colName = $options[ColOption::NAME] ?? $key;
    $table->addColumn($colName, ColumnFactory::create($database, $colName, $options));
}
```

Behavior:
- `#[Column(name: 'created_at')]` → DB column `created_at` (explicit name honored)
- `#[Column(type: DataType::INT)]` (no name) → DB column derived from property via `propertyToKey()` (backward compatible)

## Alternatives Considered

- **Always escape column identifiers in generated SQL** — would fix the MSSQL bracket issue but doesn't address the naming mismatch between what developers specify and what gets created.
- **Change `propertyToKey()` to use underscores instead of hyphens** — broader change that affects all implicit naming. Could break existing schemas.
- **Require schema loading in repositories** — workaround (what we did), not a fix.

## Consequences

- **Fixes the root cause** of Finding #3 (repository schema loading requirement) — with correct column names, SQL generation works without needing the schema loaded for escaping.
- **Breaking for existing schemas** that specify a `name` parameter but rely on the implicit kebab-case (unlikely scenario — if they specified a name, they expected it to be used).
- **Non-breaking for schemas without explicit `name`** — `propertyToKey()` still applies as default.
- **Aligns with industry standards** — developers can use `snake_case` column names as expected.

GitHub Issue: https://github.com/WebFiori/database/issues/159

## Related

- [ADR-0005](0005-request-processor-replaces-manager.md) — once column names are correct, the schema-loading workaround in repositories becomes unnecessary.
