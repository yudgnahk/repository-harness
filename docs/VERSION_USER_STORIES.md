# Version & Installation User Stories

This document breaks the versioning and installation proposals into concrete
user stories that can be contributed as individual PRs.

## Vision Alignment

The project vision is:

> Coding agents do not only need better prompts. They need better repositories.

These stories support that vision by ensuring:

- **Installed harnesses stay current** — agents get improved context over time
  without manual intervention.
- **Harness files don't collide with project files** — the harness is a clean
  layer on top of the user's project, not a invasive modification.
- **Updates are safe** — agents can evolve the harness without breaking ongoing
  work.

These stories map to existing maturity responsibilities:

| Story | Responsibility | Maturity Level |
|-------|---------------|----------------|
| US-025 Version metadata | Entropy auditing | H5 |
| US-026 Outdated check | Entropy auditing | H5 |
| US-027 Update command | Tool access, Permissions | H5 |
| US-028 Database backup | Verification | H4 |
| US-029 Schema dry-run | Verification | H4 |
| US-030 .harness/ layout | Tool access | H2 |
| US-031 Breaking change gate | Verification, Permissions | H4 |

---

## US-025: Version Metadata File

**Lane:** Normal

**Background:**

When a user installs Harness into a project, no version metadata is recorded.
There is no way to know what version is installed without reading the CLI
binary's `--version` output, and no way to know when the installation happened
or how it was installed.

**Reason:**

Entropy auditing (H5) requires knowing the harness state. Version metadata is
the foundation for all other versioning features. Without it, `outdated` and
`update` cannot function.

**Solution:**

1. New file `.harness/metadata.json` written by the installer to the target
   project root.
2. Schema:

```json
{
  "version": "0.1.10",
  "installed_at": "2026-06-19T10:30:00Z",
  "install_mode": "fresh|merge|override",
  "cli_path": "scripts/bin/harness-cli",
  "schema_version": 5
}
```

3. New command `harness-cli version` that reads `.harness/metadata.json` and
   prints:

```
CLI:           0.1.10
Installed:     0.1.10
Schema:        5
Last install:  2026-06-19 10:30 UTC (fresh)
```

4. If `.harness/metadata.json` is missing, print a warning suggesting
   re-running the installer.
5. Installer writes `.harness/metadata.json` on every install (fresh, merge,
   or override).

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `.harness/metadata.json` exists after a fresh install. | Run installer on empty directory. File exists with correct schema. |
| 2 | `harness-cli version` prints CLI version, installed version, schema version, and install timestamp. | Run command. Output matches expected format. |
| 3 | Missing metadata file produces a warning. | Delete `.harness/metadata.json`. Run `harness-cli version`. Warning printed. |
| 4 | `install_mode` reflects the installer flag used. | Run with `--merge`. Metadata shows `"install_mode": "merge"`. |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 2-3 hours

**PR scope:** Installer changes + new `version` command + metadata schema.

---

## US-026: Outdated Check

**Lane:** Tiny

**Background:**

After US-025, the installed project knows its version. But there is no way to
check whether a newer version is available without manually visiting GitHub.

**Reason:**

Entropy auditing (H5) should surface version staleness. The `audit` command
already detects data drift; it should also detect infrastructure drift (outdated
harness).

**Solution:**

1. New command `harness-cli outdated` that:
   - Reads `.harness/metadata.json` for installed version.
   - Fetches `harness-cli-release-tag` from the upstream repo (or GitHub
     releases API) for the latest version.
   - Compares and prints:

```
Installed:  0.1.10
Latest:     0.1.12
Status:     OUTDATED

Run: curl -fsSL "https://raw.githubusercontent.com/..." | bash -s -- --merge --yes
```

2. If up to date:

```
Installed:  0.1.12
Latest:     0.1.12
Status:     CURRENT
```

3. If metadata missing: print error and suggest `harness-cli version`.
4. Network failure: print warning, suggest manual check.

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `harness-cli outdated` prints installed and latest versions. | Set metadata to an old version. Run command. Both versions shown. |
| 2 | Status is `OUTDATED` when behind, `CURRENT` when current. | Test both cases. |
| 3 | Missing metadata produces a clear error. | Delete metadata. Run command. Error suggests `harness-cli version`. |
| 4 | Network failure produces a warning, not a crash. | Disconnect network. Run command. Warning printed, exit 0. |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 1-2 hours

**PR scope:** New `outdated` command. Depends on US-025.

---

## US-027: Database Backup Before Migration

**Lane:** Tiny

**Background:**

If `harness-cli migrate` fails, the database may be left in an inconsistent
state. There is no automatic backup. The user must manually copy `harness.db`
before migrating.

**Reason:**

Breaking change protection requires that every schema mutation is reversible.
A backup before migration is the simplest safety net.

**Solution:**

1. Before applying any migration, copy `harness.db` to `harness.db.bak`.
2. `harness.db.bak` is overwritten on each migration (only the last good state
   is kept).
3. New command `harness-cli migrate --repair` that restores `harness.db` from
   `harness.db.bak`.
4. If `harness.db.bak` does not exist, `--repair` prints an error.

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `harness.db.bak` exists after `harness-cli migrate`. | Run init, then migrate. Backup file exists. |
| 2 | `harness.db.bak` is a valid SQLite database. | `sqlite3 harness.db.bak "SELECT * FROM schema_version"` works. |
| 3 | `harness-cli migrate --repair` restores from backup. | Corrupt `harness.db`, run `--repair`. Database is restored. |
| 4 | `--repair` without backup prints error. | Delete `harness.db.bak`. Run `--repair`. Error printed. |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 1-2 hours

**PR scope:** Migration infrastructure changes. No schema change.

---

## US-028: Schema Dry-Run

**Lane:** Tiny

**Background:**

`harness-cli migrate` applies migrations immediately. There is no way to
preview what will change before committing.

**Reason:**

Users should know what schema changes are coming before they happen. This is
especially important for breaking change protection — a dry-run lets users
inspect risky migrations.

**Solution:**

1. Add `--dry-run` flag to `harness-cli migrate`.
2. When set, print each migration that would be applied without executing:

```
$ harness-cli migrate --dry-run
Migration 006: ALTER TABLE story ADD COLUMN priority TEXT DEFAULT 'normal';
Migration 007: CREATE TABLE story_dependency (...);
Would apply 2 migrations (schema 5 -> 7).
```

3. Exit code 0 (no side effects).

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `--dry-run` prints pending migrations without applying them. | Run with pending migrations. Output shows SQL. Database unchanged. |
| 2 | Exit code is 0. | `harness-cli migrate --dry-run; echo $?` outputs `0`. |
| 3 | No schema_version row is inserted. | Query schema_version. No new row. |
| 4 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 1 hour

**PR scope:** Migration infrastructure changes. Depends on US-027.

---

## US-029: CLI Version Gate

**Lane:** Normal

**Background:**

The CLI binary and schema version can drift. A user might download CLI v0.2.0
(expects schema v7) but only have schema v3 applied. The CLI crashes with
confusing SQL errors.

**Reason:**

Breaking change protection requires clear error messages when versions are
incompatible. This prevents data corruption from mismatched CLI and schema.

**Solution:**

1. Add constants to the CLI:

```rust
pub const MINIMUM_SCHEMA_VERSION: i64 = 5;
pub const MAXIMUM_SCHEMA_VERSION: i64 = 7;
```

2. At startup (before any command), check the database schema version against
   these constants.
3. If schema is too old:

```
Error: This CLI (v0.2.0) requires schema version 5-7.
       Current database has schema version 3.
       Run: harness-cli migrate
```

4. If schema is too new (downgraded CLI):

```
Error: This CLI (v0.1.10) requires schema version 5-7.
       Current database has schema version 8.
       This may indicate a CLI downgrade. Install the latest CLI.
```

5. Commands that don't touch the database (like `--help`, `--version`) skip
   the check.

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | CLI exits with clear error when schema version is below minimum. | Create DB with schema v3. Run any DB command. Error printed. |
| 2 | CLI exits with clear error when schema version is above maximum. | Manually insert schema_version v99. Run any DB command. Error printed. |
| 3 | `--help` and `--version` work without a database. | Delete harness.db. Run `--help`. Works. |
| 4 | Error message suggests the correct fix. | Error includes "Run: harness-cli migrate". |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 2-3 hours

**PR scope:** CLI startup logic. No schema change.

---

## US-030: Update Pre-Flight Check

**Lane:** Normal

**Background:**

The proposed `harness-cli update` command writes files. But there is no
pre-flight check to show what will change, warn about local modifications, or
back up the database.

**Reason:**

Breaking change protection requires that users see what an update will do
before it happens. This is the safety layer for the update flow.

**Solution:**

1. New command `harness-cli update --dry-run` that:
   - Reads `.harness/metadata.json` for current version.
   - Fetches the upstream file manifest for the target version.
   - For each file, compares hashes and classifies:

```
Pre-flight check:
  CLI version: 0.1.10 -> 0.1.12
  Schema version: 5 -> 6 (1 migration pending)
  Database backup: harness.db.update-bak

  Files to update:
    docs/HARNESS.md          (upstream changed, local unmodified) -> UPDATE
    docs/FEATURE_INTAKE.md   (upstream changed, local modified) -> CONFLICT
    .harness/schema/006-*.sql  (new) -> CREATE

  Warning: docs/FEATURE_INTAKE.md was modified locally.
  [s]kip  [o]verwrite  [d]iff  [a]bort?
```

2. Without `--dry-run`, the command actually writes files (future PR).
3. Always back up `harness.db` to `harness.db.update-bak` before writing.

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `--dry-run` shows files to update without writing. | Run command. No files changed. |
| 2 | Locally modified files are flagged as CONFLICT. | Edit a harness file. Run command. File flagged. |
| 3 | New upstream files are flagged as CREATE. | Check output for new files. |
| 4 | Database backup is created. | `harness.db.update-bak` exists after run. |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 3-4 hours

**PR scope:** New `update` command (dry-run only). Depends on US-025.

---

## US-031: .harness/ Layout (Future)

**Lane:** High-risk

**Background:**

The current installer scatters harness files across `docs/`, `scripts/`, and
the project root. These paths conflict with common project files.

**Reason:**

Clean separation of harness infrastructure from user project files eliminates
conflicts and makes updates deterministic. This is a breaking change to the
installation model.

**Note:** This story is **blocked** on US-025 through US-030. It should not be
attempted until version tracking and update safety are in place. The
`.harness/` layout is the end state; the current stories build the foundation.

**Solution:**

1. Move all harness-owned files into `.harness/`:

```
.harness/
  AGENTS.md         (thin shim at root stays)
  metadata.json     (version tracking)
  docs/             (harness docs)
  schema/           (migrations)
  bin/              (CLI binary)
```

2. Installer detects v1 vs v2 layout and offers migration.
3. `harness-cli migrate-layout` moves files from v1 to v2.
4. Agent shims at root (`AGENTS.md`, `CLAUDE.md`) point to `.harness/` paths.

**Acceptance Criteria:**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | Fresh install creates `.harness/` directory with all harness files. | Run installer on empty directory. `.harness/` exists. |
| 2 | User's `docs/`, `scripts/`, `AGENTS.md`, `README.md` are untouched. | Create user files before install. They remain unchanged. |
| 3 | `harness-cli migrate-layout` moves v1 files to v2. | Install v1, run migrate. Files moved. |
| 4 | Agent shim at root points to `.harness/` paths. | Read `AGENTS.md`. References `.harness/docs/`. |
| 5 | `cargo test` passes. | Run `cargo test`. |

**Estimated effort:** 6-8 hours

**PR scope:** Installer rewrite + migration script + agent shim updates.
Depends on US-025 through US-030.

---

## PR Dependency Graph

```
US-025 Version Metadata (foundation)
  │
  ├──> US-026 Outdated Check
  │
  └──> US-030 Update Pre-Flight

US-027 Database Backup
  │
  └──> US-028 Schema Dry-Run

US-029 CLI Version Gate (independent)

US-031 .harness/ Layout
  │
  └──> Depends on US-025, US-027, US-028, US-029, US-030
```

## Implementation Order

| Order | Story | Lane | Effort | PR Size |
|-------|-------|------|--------|---------|
| 1 | US-027 Database Backup | Tiny | 1-2h | Small |
| 2 | US-028 Schema Dry-Run | Tiny | 1h | Small |
| 3 | US-025 Version Metadata | Normal | 2-3h | Medium |
| 4 | US-029 CLI Version Gate | Normal | 2-3h | Medium |
| 5 | US-026 Outdated Check | Tiny | 1-2h | Small |
| 6 | US-030 Update Pre-Flight | Normal | 3-4h | Medium |
| 7 | US-031 .harness/ Layout | High-risk | 6-8h | Large |

**Total estimated effort:** 17-24 hours across 7 PRs.

## What NOT to Implement Yet

- `harness-cli update` (write mode) — depends on US-030 dry-run being validated
- `.harness/` layout — depends on all safety features being in place
- CLI deprecation protocol — premature until there's a breaking change to deprecate
