# Documentation Versioning Strategy

This document describes how documentation versions are managed in this repository.

## Branch Model

| Branch | Purpose |
|--------|---------|
| `main` | Always the **current** version's docs. All active work happens here. |
| `v3`, `v4`, etc. | Frozen snapshots created when the **next** major version's docs begin. |

## Rules

1. **`main` is always current.** Today it's v3. When v4 work starts, it becomes v4.
2. **Branch only when the next major starts.** Before making breaking v4 changes on `main`, branch `v3` off to freeze it.
3. **Never branch for minor releases.** v3.1, v3.2 docs all go on `main`. Use `> **Since X.Y**` notes.
4. **Old branches are mostly frozen.** Only typo fixes and critical corrections.
5. **Cherry-pick shared fixes.** If a fix applies to both old and current, fix on the old branch and cherry-pick to `main`.

## Timeline

```
v3.0 ships     v3.1 ships     start v4 work     v4.0 ships     start v5 work
    │               │               │                │               │
    │               │          branch v3!            │          branch v4!
    │               │               │                │               │
main: ──v3 docs────────────────────│──v4 docs───────────────────────│──v5 docs──→
                                    │                                │
v3:                                 ●───hotfixes only───────→       │
                                                                     │
v4:                                                                  ●───hotfixes──→
```

## Scenarios

### New feature in a minor release (e.g., v3.1)

Commit to `main`. Add a version note:

```markdown
## Connection Pooling

> **Since 3.1**

The library includes a built-in connection pool...
```

### Starting work on next major (e.g., v4)

```bash
git checkout main
git checkout -b v3    # Freeze v3 docs
git checkout main     # main is now v4 docs
```

### Fix a doc bug that affects both v3 and v4

```bash
git checkout v3
# apply fix
git commit -m "fix: typo in database.md"

git checkout main
git cherry-pick <commit-hash>
```

### Fix something only relevant to an old version

```bash
git checkout v3
# fix content that doesn't exist in main
git commit -m "fix: clarify deprecated syntax in v3"
# No cherry-pick needed
```

### Deprecating a feature

```markdown
> **Deprecated since 3.2.** Use `StreamingUploader` instead. Will be removed in v4.
```

When v4 docs begin on `main`, delete the deprecated content entirely. It remains in the `v3` branch.

### End of life for an old version

Keep the branch for archival (costs nothing) or delete it. Remove the old version's route from the website if you no longer want to serve it.

## Version Markers

| Situation | Marker |
|-----------|--------|
| Feature added in minor | `> **Since 3.1**` |
| Feature deprecated | `> **Deprecated since 3.2.** Use X instead.` |
| Feature removed in major | Delete from `main`; stays in old branch |
| Behavior changed in major | Update on `main`; old text stays in old branch |

## Website Routing

| URL | Source |
|-----|--------|
| `webfiori.com/docs` | `main` branch (current version) |
| `webfiori.com/docs/v3` | `v3` branch (when v4 is current) |
