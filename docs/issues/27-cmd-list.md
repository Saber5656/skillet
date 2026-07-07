# Title

skillet list: joined view of manifest, lockfile, state, and disk

## Summary

Implement `skillet list [--all-scopes]` showing, per package, the declared
range, locked version, and per-agent install status including quick drift
flags.

## Context

DESIGN 14 command table. List is the everyday observability command; it must
reconcile four sources (manifest, lock, state, filesystem existence) without
hashing by default (fast path; full hashing belongs to `verify`, issue 30).

## Scope

- `src/cli/commands/list.ts`
- `tests/cli/list.test.ts`

## Detailed Requirements

1. Synopsis: `skillet list [--all-scopes] [--scope|-g] [--agents ids…]`.
   Default: current scope only; `--all-scopes` shows project (if in one) then
   user, with section headers.
2. Row model per package (from manifest ∪ lock ∪ state):
   `{ id, spec, lockedVersion?, agents: [{ agent, status }] }` where status ∈
   `installed` (receipt exists AND dir exists) | `missing` (receipt or lock
   exists, dir absent) | `not-materialized` (in lock, no receipt for this
   agent) | `untracked` (receipt exists, package gone from manifest/lock).
3. Status derivation uses existence checks only (`fs.existsSync` on receipt
   dirs) — explicitly NO hashing (comment referencing verify).
4. Human output: table `PACKAGE  SPEC  LOCKED  <agent columns…>` with per-agent
   glyphs `ok` / `missing` / `-` / `untracked`; legend line under the table;
   packages sorted; empty scope → `no skills declared (run skillet add …)`.
5. Additionally list unmanaged skill dirs found in each selected agent's
   skills directory (dirs without receipts) under a dimmed
   `unmanaged (not touched by skillet):` section — visibility without claiming
   ownership (ADR-007 boundary).
6. `--json` data: `{ scopes: [{ scope, packages: [row…], unmanaged: [{agent,
   dir, name}] }] }`.
7. Read-only; exit 0 even when rows show `missing` (observability, not
   enforcement — verify is the enforcing command).

## Acceptance Criteria

- [ ] Fixture with all four statuses renders each glyph correctly (build via targeted state/lock manipulation in the sandbox)
- [ ] Unmanaged dir listed without any receipt created
- [ ] `--all-scopes` shows both sections; project-less cwd shows user only
- [ ] `--json` schema matches; deterministic ordering
- [ ] No hash computation occurs (spy/timing-free: assert integrity module not imported by list command via dependency-cruiser-style test or unit spy)

## Validation

`npm test -- list` green.

## Dependencies

03, 04, 09, 10, 15, 16, 17.

## Non-goals

Hash-based drift (issue 30); fixing anything (list never writes).

## Design References

- DESIGN.md 14 (command), 7.1-7.3 (sources), ADR-007
