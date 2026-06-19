# Draft Issue: Version Drift in Installed Projects

This is a draft issue to open on the upstream repo. It reports a real
problem and asks how the author handles it.

**Language:** English
**Scope:** Version drift only
**DO NOT POST until reviewed.**

---

## Title

Installed projects have no way to track or update harness version

## Body

After installing Harness into a project and later pulling upstream
improvements, I found that installed projects have no mechanism to:

1. **Record which version was installed** — no metadata file tracks the
   version or install timestamp.
2. **Check for updates** — no command compares the installed version against
   the latest release.
3. **Update incrementally** — `--merge` only creates missing files, leaving
   existing docs and templates stale. `--force` overwrites everything with
   backup, but there's no diff-aware update.

### Real example

I have two projects with Harness installed:

- **Project A** (recent install): `harness-cli 0.1.10` with `tool`,
  `audit`, `score-context`, `propose`, `intervention`, `story verify-all`.
- **Project B** (older install): `harness-cli 0.1.8` — missing all of the
  above commands. `harness-cli audit` returns
  `error: unrecognized subcommand 'audit'`.

Project B has no way to know it's behind. Running `--merge` would only add
new files (like schema 005) but not update existing docs or the CLI binary
to the current version.

### What I'm looking for

Not proposing changes to the template — just trying to understand your
workflow. How do you handle version drift in your projects? Do you:

- Track versions manually?
- Re-run with `--force` periodically?
- Something else?

Thanks!
