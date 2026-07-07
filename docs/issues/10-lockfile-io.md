# Title

core/lockfile.ts: skillet.lock read/write with byte-stable serialization

## Summary

Implement lockfile reading, validation, and atomic byte-stable writing, plus
pure helpers to upsert/remove entries and diff two lockfiles for user-facing
change tables.

## Context

DESIGN.md 7.2. The lockfile is the reproducibility contract: same inputs must
serialize to identical bytes on every OS; diffs must be reviewable line-by-line
in PRs. `install --frozen` (issue 24) and `verify` (issue 30) consume it.

## Scope

- `src/core/lockfile.ts`
- `tests/core/lockfile.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export function readLockfile(path: string): Lockfile;              // E_CONFIG on invalid; distinct message for unsupported lockfileVersion (hint: upgrade skillet)
   export function readLockfileIfExists(path: string): Lockfile | null;
   export function writeLockfile(path: string, l: Lockfile): void;    // atomic + stable via jsonio (issue 09)
   export function emptyLockfile(): Lockfile;                          // { lockfileVersion: 1, skills: {} }
   export function upsertLocked(l: Lockfile, id: string, e: LockedSkill): Lockfile; // pure
   export function removeLocked(l: Lockfile, id: string): Lockfile;    // pure, tolerant if absent
   export interface LockDiffRow { id: string; kind: "added" | "removed" | "changed";
     fromVersion?: string; toVersion?: string; fromCommit?: string; toCommit?: string; }
   export function diffLockfiles(a: Lockfile, b: Lockfile): LockDiffRow[]; // sorted by id
   ```
2. Top-level key order fixed `["lockfileVersion","skills"]`; `skills` keys
   sorted; per-entry key order fixed
   `["version","resolved","integrity","portable"]`; `resolved` key order
   `["type","url","commit","path"]` (or `["type","path"]`).
3. `lockfileVersion !== 1` → `E_CONFIG` with the found version in the message.
4. A lock entry with `resolved.type === "path"` must have `portable: false`
   (schema enforces; re-assert here with a targeted error message).
5. `diffLockfiles` reports `changed` when version, commit, path, url, or
   integrity differ (integrity-only change still reports, flagged
   `integrity changed without version change` in the row via `toVersion ===
   fromVersion` — commands render the warning).

## Acceptance Criteria

- [ ] Byte-stability: writing the DESIGN 7.2 example twice yields identical files; key order asserted literally against a snapshot
- [ ] Cross-OS: snapshot identical on windows CI (LF enforced)
- [ ] Unsupported version 2 → E_CONFIG with upgrade hint
- [ ] diff: added/removed/changed cases incl. integrity-only change
- [ ] Atomic write behavior as in issue 09

## Validation

`npm test -- lockfile` green on CI matrix.

## Dependencies

01, 02, 03, 05, 09 (shared jsonio).

## Non-goals

Resolution logic (issue 19); frozen semantics (issue 24); verify (issue 30).

## Design References

- DESIGN.md 7.2 (format), 9 (which operations touch the lock), 12.2 (lockfile trust class)
