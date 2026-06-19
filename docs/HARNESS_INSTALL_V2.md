# Harness Installation Layout v2

This document proposes relocating Harness files from scattered project-root
paths into a single `.harness/` directory, eliminating file conflicts and
enabling clean versioning.

## Current Layout (v1)

The installer copies files into the target project at these paths:

```text
project/
  AGENTS.md                          ← HIGH conflict (agent instructions)
  README.md                          ← HIGH conflict (every project has one)
  .gitignore                         ← MEDIUM conflict (append harness rules)
  docs/
    ARCHITECTURE.md                  ← HIGH conflict (common project dir)
    CONTEXT_RULES.md
    FEATURE_INTAKE.md
    GLOSSARY.md
    HARNESS.md
    HARNESS_AUDIT.md
    HARNESS_BACKLOG.md
    HARNESS_COMPONENTS.md
    HARNESS_MATURITY.md
    IMPROVEMENT_PROTOCOL.md
    README.md
    TEST_MATRIX.md
    TOOL_REGISTRY.md
    TRACE_SPEC.md
    decisions/
      0001-harness-first-development.md
      0002-post-spec-product-lifecycle.md
      0003-generic-spec-intake-harness.md
      0004-sqlite-durable-layer.md
      0005-prebuilt-rust-harness-cli.md
      0006-phase-4-benchmark-triage.md
      0007-improvement-proposal-rules.md
      README.md
    product/
      README.md
    stories/
      README.md
      backlog.md
    templates/
      decision.md
      spec-intake.md
      story.md
      validation-report.md
      high-risk-story/
        design.md
        execplan.md
        overview.md
        validation.md
  scripts/
    README.md                        ← HIGH conflict (common project dir)
    bin/
      harness-cli                    ← Binary (or .exe on Windows)
    schema/
      001-init.sql
      002-story-verify.sql
      003-tool-registry.sql
      004-intervention.sql
      005-tool-extensions.sql
```

### Conflict Analysis

| Path | Risk | Why |
|------|------|-----|
| `AGENTS.md` | **HIGH** | Many projects use this for agent instructions. Overwrites user content. |
| `README.md` | **HIGH** | Every project has one. Installer backs up then replaces. |
| `docs/` | **HIGH** | Common directory. Installer appends/merges, but structure conflicts. |
| `scripts/` | **HIGH** | Common directory. Installer appends, but users may have `scripts/test.sh` etc. |
| `.gitignore` | **MEDIUM** | Append-only, but adds harness-specific rules to user's gitignore. |

### Current Conflict Resolution

The installer offers three modes when protected paths exist:

- **Merge** (`--merge`): Skip existing files, only create missing ones. Safe but
  stale — existing files never get updated.
- **Override** (`--override`): Back up everything, then replace. Destroys user
  customizations.
- **Stop** (default): Refuse to install. User must manually resolve.

None of these are satisfying. Merge leaves you frozen. Override destroys work.
Stop requires manual intervention.

## Proposed Layout (v2)

Move all Harness-owned files into `.harness/`:

```text
project/
  .harness/
    AGENTS.md                        ← Agent shim (imported by CLAUDE.md)
    metadata.json                    ← Version tracking (see VERSION_SPEC.md)
    docs/
      ARCHITECTURE.md
      CONTEXT_RULES.md
      FEATURE_INTAKE.md
      GLOSSARY.md
      HARNESS.md
      HARNESS_AUDIT.md
      HARNESS_BACKLOG.md
      HARNESS_COMPONENTS.md
      HARNESS_MATURITY.md
      IMPROVEMENT_PROTOCOL.md
      README.md
      TEST_MATRIX.md
      TOOL_REGISTRY.md
      TRACE_SPEC.md
      decisions/
        0001-harness-first-development.md
        0002-post-spec-product-lifecycle.md
        ...
      product/
        README.md
      stories/
        README.md
        backlog.md
      templates/
        decision.md
        spec-intake.md
        story.md
        validation-report.md
        high-risk-story/
          design.md
          execplan.md
          overview.md
          validation.md
    schema/
      001-init.sql
      002-story-verify.sql
      003-tool-registry.sql
      004-intervention.sql
      005-tool-extensions.sql
    bin/
      harness-cli                    ← Binary
  scripts/
    README.md                        ← User's scripts (NOT harness)
  docs/
    README.md                        ← User's docs (NOT harness)
  AGENTS.md                          ← User's agent instructions (NOT harness)
  README.md                          ← User's project README (NOT harness)
```

### What Changes

| Item | v1 | v2 |
|------|----|----|
| Agent shim | `./AGENTS.md` (shared with user) | `.harness/AGENTS.md` (isolated) |
| Harness docs | `./docs/*.md` (shared with user) | `.harness/docs/*.md` (isolated) |
| Schema | `./scripts/schema/` (shared with user) | `.harness/schema/` (isolated) |
| CLI binary | `./scripts/bin/harness-cli` | `.harness/bin/harness-cli` |
| Metadata | None | `.harness/metadata.json` |
| User docs | Mixed with harness docs | Clean separation |
| User scripts | Mixed with harness scripts | Clean separation |

### What Stays at Root

These files must remain at the project root because agents and tools expect
them there:

- `AGENTS.md` — Becomes a **thin shim** that imports from `.harness/`:
  ```markdown
  # Agent Instructions

  Add project-specific agent instructions here.

  <!-- HARNESS:BEGIN -->
  ## Harness

  This repo uses Harness. Before work, read:

  - `.harness/docs/HARNESS.md`
  - `.harness/docs/FEATURE_INTAKE.md`
  - `.harness/docs/ARCHITECTURE.md`
  - `.harness/docs/CONTEXT_RULES.md`
  - `.harness/docs/TOOL_REGISTRY.md`
  - `.harness/bin/harness-cli query matrix`
  <!-- HARNESS:END -->
  ```

- `CLAUDE.md` — Optional shim for Claude Code (same pattern as today, but
  pointing to `.harness/` paths).

### Why `.harness/` Not `.harness-db/` or `harness/`?

| Candidate | Pros | Cons |
|-----------|------|------|
| `.harness/` | Hidden by default (like `.github/`), convention for tool config | Some users hide dotfiles |
| `harness/` | Visible, clear | Conflicts with project directories named "harness" |
| `.harness-db/` | Matches existing `.harness-backup/` | Too narrow — it's not just the DB |
| `harness-config/` | Descriptive | Verbose, not a convention |

`.harness/` wins because:
1. Follows the `.github/`, `.vscode/`, `.idea/` convention.
2. Hidden by default in file explorers and `ls`.
3. Clearly owned by the Harness tool.
4. The backup directory `.harness-backup/` stays adjacent.

## Migration Strategy

### Phase 1: Dual-Mode Installer

The installer detects which layout the target uses:

```text
Check for:
  1. .harness/metadata.json  → v2 layout (update in place)
  2. docs/HARNESS.md          → v1 layout (offer migration)
  3. Neither                  → fresh install (use v2)
```

For v1 projects, the installer offers:

```text
Harness v1 layout detected. Migrate to v2 (.harness/)?

  1. Migrate  Move harness files into .harness/, leave user files
  2. Install v2 alongside v1 (warns about duplicates)
  3. Skip     Keep v1 layout
```

### Phase 2: Migration Script

```bash
harness-cli migrate-layout
```

This command:

1. Reads the v1 file list from the installer manifest.
2. For each file:
   - If file exists at `./path` and is unmodified (matches upstream hash):
     move to `.harness/path`.
   - If file exists and was modified by user:
     keep at `./path`, copy upstream version to `.harness/path`,
     warn about the conflict.
   - If file doesn't exist: create at `.harness/path`.
3. Write `.harness/metadata.json`.
4. Update the agent shim in `./AGENTS.md` to point to `.harness/` paths.
5. Print summary.

### Phase 3: Remove v1 Support

After a deprecation period, the installer stops supporting v1 layout.

## Agent Shim Design

Agents (Claude Code, Codex, Cursor) look for `AGENTS.md` at the project root.
This cannot change without breaking every agent. The solution is a **thin shim**
at the root that points into `.harness/`:

```markdown
# Agent Instructions

Add project-specific agent instructions here.

<!-- HARNESS:BEGIN -->
## Harness

This repo uses Harness. Before work, read:

- `.harness/docs/HARNESS.md`
- `.harness/docs/FEATURE_INTAKE.md`
- `.harness/docs/ARCHITECTURE.md`
- `.harness/docs/CONTEXT_RULES.md`
- `.harness/docs/TOOL_REGISTRY.md`
- `.harness/bin/harness-cli query matrix`

Use the Rust Harness CLI at `.harness/bin/harness-cli` as the main operational
tool.
<!-- HARNESS:END -->
```

The user's own agent instructions go above or below the `<!-- HARNESS:BEGIN
-->` block, exactly as today.

### CLAUDE.md Shim

```markdown
<!-- HARNESS:BEGIN -->
## Harness

Claude Code loads this file into every session, but it does not auto-load
`AGENTS.md`. The bare `@` lines below import the always-required harness
context at context-load time.

@.harness/AGENTS.md

@.harness/docs/FEATURE_INTAKE.md

Also run `.harness/bin/harness-cli query matrix` before starting work.
<!-- HARNESS:END -->
```

## Impact on Existing Commands

| Command | v1 Path | v2 Path |
|---------|---------|---------|
| `harness-cli` binary | `scripts/bin/harness-cli` | `.harness/bin/harness-cli` |
| `harness-cli init` | Creates `harness.db` in cwd | Same (DB stays at root) |
| `harness-cli migrate` | Reads `scripts/schema/` | Reads `.harness/schema/` |
| `harness-cli story add` | Writes to `harness.db` | Same |
| Agent shim | `AGENTS.md` at root | `AGENTS.md` at root (thin shim) |

**Important:** The database (`harness.db`) stays at the project root. It is
the project's durable state, not Harness infrastructure. Only the tooling and
documentation move into `.harness/`.

## Benefits

1. **Zero conflicts** — `docs/`, `scripts/`, `AGENTS.md`, `README.md` are
   fully available for the user's project.
2. **Clean updates** — `harness-cli update` knows exactly which files it owns.
3. **Clear ownership** — anything in `.harness/` is Harness; everything else is
   the user's project.
4. **Version tracking** — `.harness/metadata.json` records the installed
   version and file hashes.
5. **Backup isolation** — `.harness-backup/` already exists at root; the
   `.harness/` directory is separate and clean.
6. **Discoverable** — `.harness/` is visible in file explorers and `ls -a`,
   following `.github/` conventions.

## Risks

| Risk | Mitigation |
|------|------------|
| Agent shims break if paths change | Shim is updated atomically by installer |
| User adds files inside `.harness/` | Installer warns; `.harness/` is Harness-owned |
| Breaking change for existing installs | Dual-mode installer; migration command |
| `harness.db` location confusion | DB stays at root; docs explain why |

## Open Questions

1. Should `harness.db` also move into `.harness/`? Pros: single location.
   Cons: user may want to `.gitignore` only the DB, not the whole directory.
2. Should the CLI binary be `.harness/bin/harness-cli` or
   `.harness/harness-cli` (flat)? Binary in `bin/` follows convention.
3. How long is the v1 deprecation period? 6 months? Until v1.0?
