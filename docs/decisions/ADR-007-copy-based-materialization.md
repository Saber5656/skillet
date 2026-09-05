# ADR-007: Copy-based materialization with receipt-tracked state

- Status: Accepted (conservative default, 2026-07-07)
- Deciders: Fable

## Context

Installed skills must appear inside each runtime's `skills/` directory.
Alternatives: (a) symlink from a canonical store (vercel-labs/skills default) —
single source of truth, but Windows symlink privileges, cloud-synced home
directories (iCloud/Dropbox), and runtime file-watching make symlinks the #1
portability risk; (b) copies per target — duplicated bytes, but boring and
universal.

## Decision

Copies, always. Every materialized directory is recorded in a per-scope
`state.json` receipt `{package, version, integrity, agent, dir, installedAt}`.
Mutating operations use atomic sibling-rename swaps. skillet only replaces or
deletes directories that (1) have a receipt and (2) hash-match the receipt
(`--force` relaxes only the hash check, never the receipt requirement).
Drift is a feature: `skillet verify` rehashes copies against the lockfile.

## Consequences

- Positive: works identically on macOS/Linux/Windows and synced filesystems;
  a user hand-editing an installed skill is detected (drift), not silently
  propagated; uninstall can never eat user-authored skills.
- Negative: N agents × M skills copies on disk (skills are ≤ 20 MiB by policy —
  acceptable); updates must re-materialize every target.
