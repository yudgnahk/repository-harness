# Version Specification

This document defines how `repository-harness` tracks, communicates, and updates
its version across installed projects.

## Problem

When a user installs Harness into a project, no version metadata is recorded.
The installer has no way to know what version is present, and there is no
mechanism to update an existing installation. Over time:

- Installed docs and templates drift behind upstream improvements.
- Schema migrations accumulate without the user knowing which are applied.
- The CLI binary may be outdated, missing new commands or bug fixes.
- The `audit` command detects data drift but not infrastructure staleness.

## Goals

1. Every installed project knows its Harness version.
2. Users can check whether their installation is outdated.
3. Updates are safe, incremental, and preserve user customizations.
4. The CLI can enforce minimum version requirements for features.

## Version Format

Harness uses **semantic versioning** aligned with the CLI release tag:

```
harness-cli-v0.1.10
       │      │  │  │
       │      │  │  └─ patch: bug fixes, schema migrations, doc tweaks
       │      │  └──── minor: new commands, new docs, backward-compatible
       │      └─────── major: breaking changes (schema restructure, CLI rework)
       └────────────── prefix (always "harness-cli-v")
```

The canonical version source is `scripts/harness-cli-release-tag` in the
upstream repository. The CLI binary reports its version via `--version`.

## Metadata File

### Location

`.harness/metadata.json` in the target project root.

### Schema

```json
{
  "version": "0.1.10",
  "installed_at": "2026-06-19T10:30:00Z",
  "install_mode": "fresh|merge|override",
  "cli_path": "scripts/bin/harness-cli",
  "schema_version": 5,
  "files": {
    "docs/HARNESS.md": "a1b2c3d4",
    "docs/FEATURE_INTAKE.md": "e5f6g7h8"
  }
}
```

| Field | Purpose |
|-------|---------|
| `version` | Harness version at install time. Read from `harness-cli-release-tag`. |
| `installed_at` | ISO 8601 timestamp of last install or update. |
| `install_mode` | How the installer was invoked: `fresh`, `merge`, or `override`. |
| `cli_path` | Relative path to the CLI binary. |
| `schema_version` | Highest applied migration number (e.g., 5 for `005-tool-extensions.sql`). |
| `files` | Map of installed file paths to content hashes (optional, for diff-aware updates). |

### Why `schema_version`?

The CLI already tracks applied migrations in the `schema_version` table inside
`harness.db`. But the metadata file provides a quick check without opening the
database. The CLI keeps both in sync.

### Why content hashes?

The `files` map enables diff-aware updates. When the user runs `harness-cli
update`, the installer can compare upstream hashes against installed hashes to
determine which files actually changed. Files the user has edited locally are
detected by comparing the on-disk hash against the recorded hash.

## Commands

### `harness-cli --version`

Already exists. Output:

```
harness-cli 0.1.10
```

### `harness-cli version`

New command. Shows both CLI and installed harness metadata:

```
harness-cli version
CLI:           0.1.10
Installed:     0.1.10
Schema:        5
Last install:  2026-06-19 10:30 UTC (fresh)
```

If `.harness/metadata.json` is missing, the command prints a warning and
suggests re-running the installer.

### `harness-cli outdated`

New command. Compares installed version against the latest GitHub release.

```
harness-cli outdated
Installed:  0.1.10
Latest:     0.1.12
Status:     OUTDATED

Run: curl -fsSL "https://raw.githubusercontent.com/hoangnb24/repository-harness/main/scripts/install-harness.sh" | bash -s -- --merge --yes
```

If up to date:

```
harness-cli outdated
Installed:  0.1.12
Latest:     0.1.12
Status:     CURRENT
```

**Implementation:** Fetches
`https://github.com/hoangnb24/repository-harness/releases/latest/download/harness-cli-release-tag`
and compares against `.harness/metadata.json`.

### `harness-cli update`

New command. Updates harness files in the current project.

```
harness-cli update [OPTIONS]

Options:
  --dry-run       Show what would change without writing
  --force         Overwrite files even if locally modified
  --yes           Accept defaults and skip prompts
  --from <ref>    Update from a specific git ref or release tag
```

**Behavior:**

1. Read `.harness/metadata.json` for current version.
2. Fetch the file manifest from the target version (upstream `install-harness.sh`
   file list or a generated manifest).
3. For each file:
   - If file does not exist locally: **create** (new in upstream).
   - If file exists and hashes match: **skip** (unchanged).
   - If file exists and only upstream changed: **update** (safe).
   - If file exists and local was also modified: **conflict** (ask user).
4. Update `.harness/metadata.json` with new version and hashes.
5. Run `harness-cli migrate` if schema changed.

**Conflict resolution:**

```text
Conflict: docs/ARCHITECTURE.md
  Local hash:   a1b2c3d4
  Upstream hash: e5f6g7h8

  [s]kip   - keep local version
  [o]verwrite - replace with upstream (backup first)
  [m]erge  - attempt diff3 merge (future)
  [a]bort  - stop update
```

## Update Flow

```
User runs: harness-cli update
  │
  ├─ 1. Read .harness/metadata.json (current version)
  │
  ├─ 2. Fetch latest version from GitHub
  │     └─ Compare: 0.1.10 -> 0.1.12
  │
  ├─ 3. Download file manifest for 0.1.12
  │     └─ List of files + content hashes
  │
  ├─ 4. Diff against installed files
  │     ├─ New files:       create
  │     ├─ Unchanged:       skip
  │     ├─ Upstream-only:   update (safe)
  │     └─ Both modified:   conflict (interactive)
  │
  ├─ 5. Download and write updated files
  │
  ├─ 6. Run schema migrations if needed
  │
  └─ 7. Update .harness/metadata.json
```

## Audit Integration

Add a check to `harness-cli audit`:

```
Harness version: 0.1.10 (latest: 0.1.12) — STALE
  Run: harness-cli outdated
```

This appears in the audit output alongside drift categories. The entropy score
does not penalize version staleness (it is informational, not data drift).

## Backward Compatibility

- Projects installed before this spec exist will not have
  `.harness/metadata.json`. The `version` and `outdated` commands detect this
  and suggest re-running the installer.
- The CLI binary version (from `--version`) remains the primary identifier. The
  metadata file is supplementary.
- The `--merge` installer flag continues to work as before. The metadata file
  is always written, regardless of install mode.

## Safety Guarantees

Updates must never break an installed project. The full protection model is
defined in `docs/BREAKING_CHANGE_PROTECTION.md`. Summary:

- **Schema migrations run in transactions.** If a migration fails, the database
  rolls back to its previous state.
- **Database is backed up before every migration.** `harness.db.bak` is
  overwritten each time. A failed migration can be rolled back with
  `harness-cli migrate --repair`.
- **CLI checks schema compatibility at startup.** If the CLI expects schema v7
  but the database has v3, it prints a clear error and tells the user to run
  `harness-cli migrate`.
- **Schema dry-run before apply.** `harness-cli migrate --dry-run` shows what
  would change without writing.
- **Update pre-flight checks.** Before writing files, `harness-cli update`
  shows what will change, warns about local modifications, and backs up the
  database.
- **Migrations are backward-compatible within a major version.** A v0.1.x CLI
  can read a database migrated by v0.2.0. Breaking schema changes require a
  major version bump.

## Migration Path

1. **Phase:** Add `version` command and `.harness/metadata.json` writing to the
   installer. (Normal lane)
2. **Phase:** Add `outdated` command. (Tiny lane)
3. **Phase:** Add `update` command with diff-aware logic. (High-risk lane)
4. **Phase:** Add audit integration. (Tiny lane)
