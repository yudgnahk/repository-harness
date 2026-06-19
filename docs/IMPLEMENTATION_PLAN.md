# Implementation Plan: Version Tracking Features

This document breaks down the implementation into concrete tasks for the
`feat/version-tracking` branch.

## Implementation Order

| Order | Story | Lane | Effort | Files to Change |
|-------|-------|------|--------|-----------------|
| 1 | US-027: Database Backup | Tiny | 1-2h | `infrastructure.rs` |
| 2 | US-028: Schema Dry-Run | Tiny | 1h | `interface.rs`, `infrastructure.rs` |
| 3 | US-029: CLI Version Gate | Normal | 2-3h | `infrastructure.rs`, `main.rs` |
| 4 | US-025: Version Metadata | Normal | 2-3h | `interface.rs`, `application.rs`, `infrastructure.rs`, installer |
| 5 | US-026: Outdated Check | Tiny | 1-2h | `interface.rs`, `application.rs`, `infrastructure.rs` |

## Task Breakdown

### US-027: Database Backup Before Migration

**Files to change:**
- `crates/harness-cli/src/infrastructure.rs` — add backup logic to `migrate()`

**Concrete steps:**
1. In `migrate()`, before calling `apply_pending_migrations()`:
   - Check if `harness.db` exists
   - Copy `harness.db` to `harness.db.bak`
   - Log: "Backup created: harness.db.bak"
2. Add `--repair` flag to `Migrate` command in `interface.rs`
3. In `migrate()`, if `--repair`:
   - Check if `harness.db.bak` exists
   - If not, print error and return
   - Copy `harness.db.bak` to `harness.db`
   - Log: "Database restored from harness.db.bak"
4. Add tests for backup, repair, and missing backup cases

**Test cases:**
- `test_migrate_creates_backup` — run migrate, verify harness.db.bak exists
- `test_migrate_repair_restores` — corrupt db, run --repair, verify restored
- `test_migrate_repair_no_backup` — delete backup, run --repair, verify error

---

### US-028: Schema Dry-Run

**Files to change:**
- `crates/harness-cli/src/interface.rs` — add `--dry-run` flag to `Migrate`
- `crates/harness-cli/src/infrastructure.rs` — add dry-run logic

**Concrete steps:**
1. Add `dry_run: bool` field to `MigrateArgs` in `interface.rs`
2. In `migrate()`, if `dry_run`:
   - Call `migration_files()` to get pending migrations
   - For each pending migration, print: "Migration NNN: <first line of SQL>"
   - Print summary: "Would apply N migrations (schema X -> Y)"
   - Return without applying
3. Add tests for dry-run behavior

**Test cases:**
- `test_migrate_dry_run_shows_pending` — run with --dry-run, verify output
- `test_migrate_dry_run_no_side_effects` — run with --dry-run, verify db unchanged
- `test_migrate_dry_run_exit_code` — run with --dry-run, verify exit 0

---

### US-029: CLI Version Gate

**Files to change:**
- `crates/harness-cli/src/infrastructure.rs` — add version constants and check

**Concrete steps:**
1. Add constants at top of `infrastructure.rs`:
   ```rust
   pub const MINIMUM_SCHEMA_VERSION: i64 = 5;
   pub const MAXIMUM_SCHEMA_VERSION: i64 = 7;
   ```
2. Add `check_schema_version()` function:
   - Read current schema version from database
   - If version < MINIMUM: print error "This CLI requires schema >= {MIN}. Run: harness-cli migrate"
   - If version > MAXIMUM: print error "This CLI requires schema <= {MAX}. CLI may be outdated."
   - Return Result
3. Call `check_schema_version()` in `open_existing()` (before every DB operation)
4. Skip check for commands that don't touch DB: `--help`, `--version`, `init`
5. Add tests for version gate

**Test cases:**
- `test_version_gate_too_old` — create db with schema v3, verify error
- `test_version_gate_too_new` — manually insert schema v99, verify error
- `test_version_gate_ok` — normal db, verify no error
- `test_version_gate_skips_init` — verify init works without existing db

---

### US-025: Version Metadata

**Files to change:**
- `crates/harness-cli/src/interface.rs` — add `Version` command
- `crates/harness-cli/src/application.rs` — add `get_version_info()` method
- `crates/harness-cli/src/infrastructure.rs` — add metadata read/write
- `scripts/install-harness.sh` — write `.harness/metadata.json` on install

**Concrete steps:**
1. Create `.harness/metadata.json` schema in design doc (already done)
2. Add `Version` command to `Command` enum in `interface.rs`
3. Add `VersionInfo` struct to `application.rs`
4. Add `get_version_info()` to `HarnessService`:
   - Read `.harness/metadata.json` if exists
   - Read CLI version from `--version`
   - Read schema version from database
   - Return combined info
5. Add `write_metadata()` to `infrastructure.rs`:
   - Called after successful `init()` and `migrate()`
   - Writes/updates `.harness/metadata.json`
6. Update `install-harness.sh`:
   - After installing files, create `.harness/metadata.json`
   - Read version from `harness-cli-release-tag`
7. Add tests

**Test cases:**
- `test_version_command_shows_info` — run version, verify output
- `test_version_missing_metadata` — delete metadata, verify warning
- `test_metadata_written_on_init` — run init, verify metadata exists
- `test_metadata_written_on_migrate` — run migrate, verify metadata updated

---

### US-026: Outdated Check

**Files to change:**
- `crates/harness-cli/src/interface.rs` — add `Outdated` command
- `crates/harness-cli/src/application.rs` — add `check_outdated()` method
- `crates/harness-cli/src/infrastructure.rs` — add GitHub API fetch

**Concrete steps:**
1. Add `Outdated` command to `Command` enum in `interface.rs`
2. Add `OutdatedResult` struct to `application.rs`
3. Add `check_outdated()` to `HarnessService`:
   - Read installed version from `.harness/metadata.json`
   - Fetch latest version from GitHub releases API
   - Compare and return result
4. Add `fetch_latest_version()` to `infrastructure.rs`:
   - Fetch `https://api.github.com/repos/hoangnb24/repository-harness/releases/latest`
   - Parse JSON, extract tag name
   - Handle network errors gracefully
5. Add tests

**Test cases:**
- `test_outdated_shows_versions` — set old metadata, verify output
- `test_outdated_current` — set current metadata, verify CURRENT
- `test_outdated_missing_metadata` — delete metadata, verify error
- `test_outdated_network_failure` — mock network error, verify warning

---

## Dependencies

```
US-027 Database Backup
  └─> US-028 Schema Dry-Run

US-029 CLI Version Gate (independent)

US-025 Version Metadata
  └─> US-026 Outdated Check
```

## Testing Strategy

All changes must pass:
1. `cargo test` — all existing + new tests pass
2. `cargo clippy` — no warnings
3. `cargo fmt --check` — formatting correct
4. Manual test on `worldcup-magazine` project (v0.1.8) to verify backward compatibility

## What NOT to Implement Yet

- `harness-cli update` (write mode) — depends on US-030 dry-run being validated
- `.harness/` layout — depends on all safety features being in place
- CLI deprecation protocol — premature until there's a breaking change to deprecate
