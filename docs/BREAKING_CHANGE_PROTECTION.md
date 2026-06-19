# Breaking Change Protection

This document defines how `repository-harness` prevents installed projects from
being broken by upstream updates.

## The Risk

When a user runs `harness-cli update` (or re-runs the installer), the new
version might:

1. **Change CLI command signatures** — flags renamed, output format changed,
   exit codes changed
2. **Break schema migrations** — new migration assumes columns that don't exist,
   or drops columns that existing queries use
3. **Corrupt the database** — partial migration leaves `harness.db` in an
   inconsistent state
4. **Break agent shims** — paths in `AGENTS.md` or `CLAUDE.md` change, agents
   can't find docs
5. **Break downstream tools** — scripts or CI that call `harness-cli` break on
   new output format

## Current Safeguards

| Safeguard | Status | Gap |
|-----------|--------|-----|
| Schema migration is forward-only | ✅ Exists | No rollback, no dry-run |
| CLI `--version` reports version | ✅ Exists | No minimum version check |
| Installer backs up before overwrite | ✅ Exists | No content-aware backup |
| Migrations are additive (ALTER TABLE ADD) | ✅ Exists | No DROP TABLE/COLUMN yet |
| No CLI command compatibility check | ❌ Missing | New CLI might fail on old data |

## What's Missing

### 1. No Schema Rollback

If migration 006 fails (e.g., a column already exists from a partial previous
run), the database is stuck. There is no `harness-cli migrate rollback` or
`harness-cli migrate repair`.

### 2. No CLI Version Gate

The CLI binary and the schema version can drift apart. A user might download
CLI v0.2.0 (expects schema v7) but only have schema v3 applied. The CLI will
crash with confusing SQL errors instead of a clear message.

### 3. No Update Dry-Run

`harness-cli update` (proposed) would write files without showing the user
what's about to change. A bad update could overwrite customized files.

### 4. No Database Backup Before Update

If an update changes schema and the migration fails, the original database is
gone. The installer backs up files but not the database.

### 5. No Compatibility Matrix

There is no document that says "CLI v0.1.10 requires schema v5" or "CLI
v0.2.0 breaks CLI v0.1.x output format."

## Proposed Protections

### A. Version Compatibility Table

Maintain a compatibility matrix in the source repo:

```text
// crates/harness-cli/src/compat.rs

pub const MINIMUM_SCHEMA_VERSION: i64 = 5;
pub const MAXIMUM_SCHEMA_VERSION: i64 = 7;
pub const CLI_VERSION: &str = "0.2.0";
```

The CLI checks at startup:

```
$ harness-cli story add ...
Error: This CLI (v0.2.0) requires schema version 5-7.
       Current database has schema version 3.
       Run: harness-cli migrate
```

### B. Schema Dry-Run

Add `--dry-run` to `harness-cli migrate`:

```
$ harness-cli migrate --dry-run
Migration 006: ALTER TABLE story ADD COLUMN priority TEXT DEFAULT 'normal';
Migration 007: CREATE TABLE story_dependency (...);
Would apply 2 migrations (schema 5 -> 7).
```

This lets users preview schema changes before committing.

### C. Database Backup Before Migration

Every migration automatically creates a backup:

```text
harness.db          ← active database
harness.db.bak      ← backup from last migration (overwritten each time)
```

If migration fails, the user can restore:

```
$ harness-cli migrate --repair
Migration 006 failed. Restored harness.db from harness.db.bak.
```

### D. Migration Transaction Safety

Each migration runs inside a SQLite transaction. If any statement fails,
the entire migration rolls back:

```sql
BEGIN TRANSACTION;
ALTER TABLE story ADD COLUMN priority TEXT DEFAULT 'normal';
-- if this fails, the ADD COLUMN above rolls back
INSERT INTO schema_version (version) VALUES (6);
COMMIT;
```

The current code already uses `execute_batch` which wraps in a transaction
for DDL in SQLite. But this should be explicit and documented.

### E. Update Pre-Flight Check

Before `harness-cli update` writes anything:

1. **Check schema compatibility** — will the new CLI work with the current
   schema? If not, warn and offer to migrate first.
2. **Check for local modifications** — compare file hashes. If the user
   modified a file, warn and ask.
3. **Create database backup** — copy `harness.db` to `harness.db.update-bak`.
4. **Dry-run the update** — show what files will change.
5. **Ask for confirmation** — unless `--yes` is passed.

```text
$ harness-cli update
Pre-flight check:
  CLI version: 0.1.10 -> 0.2.0
  Schema version: 5 -> 7 (2 migrations pending)
  Database backup: harness.db.update-bak

  Files to update:
    docs/HARNESS.md          (upstream changed, local unmodified) -> UPDATE
    docs/FEATURE_INTAKE.md   (upstream changed, local modified) -> CONFLICT
    .harness/schema/006-*.sql  (new) -> CREATE

  Warning: docs/FEATURE_INTAKE.md was modified locally.
  [s]kip  [o]verwrite  [d]iff  [a]bort?
```

### F. CLI Command Deprecation Protocol

When a command changes behavior:

1. **v1 (current)** — Command works as before. Print deprecation warning:
   ```
   Warning: --format is deprecated. Use --output instead.
   ```
2. **v2 (next)** — Command accepts both old and new flags. Old flags still
   trigger deprecation warning.
3. **v3 (future)** — Old flags are removed.

This gives users two release cycles to migrate.

### G. Schema Migration Versioning Contract

| Change Type | Migration Rule |
|-------------|----------------|
| ADD COLUMN | Safe. Existing rows get default value. |
| ADD TABLE | Safe. No existing data affected. |
| ADD INDEX | Safe. Builds on existing data. |
| DROP COLUMN | **BREAKING.** Must go through deprecation: first make nullable, then stop reading, then drop in next major version. |
| DROP TABLE | **BREAKING.** Must go through deprecation cycle. |
| RENAME COLUMN | **BREAKING.** Never do directly. Add new column, backfill, switch reads, drop old. |
| CHANGE TYPE | **BREAKING.** Add new column, backfill, switch reads, drop old. |

**Rule: Migrations must be backward-compatible.** A v0.1.x CLI must be able to
read a database migrated by v0.2.0 (within the same major version).

### H. Installer Backup Scope

The installer currently backs up files. It should also back up the database:

```text
.harness-backup/
  20260619103000/
    AGENTS.md
    docs/HARNESS.md
    scripts/bin/harness-cli
    harness.db          ← NEW: database backup
    metadata.json       ← NEW: previous metadata
```

## Migration Failure Recovery

### Scenario 1: Migration 006 Fails Mid-Way

```text
$ harness-cli migrate
Applying migration 006...
Error: column priority: duplicate column name

Recovery options:
  1. Restore from backup: harness-cli migrate --repair
  2. Fix manually: open harness.db with sqlite3
  3. Skip: harness-cli migrate --skip 006
```

### Scenario 2: New CLI Can't Read Old Schema

```text
$ harness-cli story add --title "Test" --lane normal
Error: This CLI requires schema version >= 5.
       Current database: version 3.
       Run: harness-cli migrate
```

### Scenario 3: Update Breaks Agent Shim

```text
$ harness-cli update
...
Error: Updated AGENTS.md shim references .harness/docs/HARNESS.md
       but .harness/ directory does not exist.
       Rollback: cp .harness-backup/.../AGENTS.md ./AGENTS.md
```

## Implementation Priority

| Protection | Effort | Risk Reduction | Priority |
|------------|--------|----------------|----------|
| Version compatibility table | Tiny | High | P0 |
| Schema dry-run | Tiny | Medium | P0 |
| Database backup before migration | Tiny | High | P0 |
| Update pre-flight check | Normal | High | P1 |
| CLI deprecation protocol | Normal | Medium | P1 |
| Migration transaction safety | Tiny | Medium | P1 |
| Update dry-run | Normal | Medium | P2 |
| Migration rollback command | Normal | Medium | P2 |

## Open Questions

1. Should `harness.db` be backed up on every `harness-cli` invocation, or only
   before migrations? Every invocation is safer but slower.
2. How many versions back should the CLI support? Same major? Same minor?
3. Should the installer refuse to install a CLI that's older than the schema
   version in the database?
4. Should `harness-cli update` automatically run `harness-cli migrate` after
   updating files? Or leave that to the user?
