# Title

skillet remove: delete packages from manifest, lock, targets, and state

## Summary

Implement `skillet remove <names…>`: unmaterialize from every recorded target
with the managed-only safety rules, then delete manifest/lock entries and
receipts.

## Context

DESIGN 9 remove row + 10.3 removal safety (ADR-007): only receipt-tracked
dirs are deleted; drifted copies require `--force`; unmanaged dirs are never
touched. Order matters: disk first, records last — a crash mid-way must leave
records that still point at reality.

## Scope

- `src/cli/commands/remove.ts`
- `tests/cli/remove.test.ts`

## Detailed Requirements

1. Synopsis: `skillet remove <names…> [--scope|-g] [--force] [--yes]`.
   Names must exist in manifest (E_UNKNOWN_PACKAGE listing near matches from
   the manifest keys, not the index).
2. Per package: collect receipts across ALL agents (state), not just currently
   selected agents — remove means remove everywhere it was recorded.
3. Execution order per package: for each receipt → `unmaterialize`
   (issue 21; drifted + no `--force` → E_TARGET_CONFLICT aborts the whole
   command before record changes, listing the drifted dirs and the exact
   `--force` invocation) → after ALL receipts of the package are gone: remove
   receipts from state (write), then lock entry, then manifest entry (each
   atomic; this order means a crash leaves extra records but no orphan files —
   matching the "records may over-claim, never under-claim" comment contract
   from issue 21).
4. No consent prompt (removal is the user's explicit intent; `--yes` accepted
   and ignored for symmetry). Confirmation IS required (y/N via issue-22
   readLine, or `--yes`) only when `--force` will overwrite drifted content —
   destroying user edits deserves a pause.
5. Human output: `- anthropics/pdf@1.2.0 removed from claude-code (project),
   codex (project)`; missing dirs noted as `(already absent)`. `--json`:
   `{ removed: [{ id, targets, alreadyAbsent }] }`.
6. Removing a package whose dirs are already gone (user deleted manually) must
   succeed and clean records (receipt hash check skipped for non-existent
   dirs — issue 21 unmaterialize contract).

## Acceptance Criteria

- [ ] Full removal: dirs gone, state empty, lock/manifest entries gone, exit 0
- [ ] Drifted copy without --force → exit 4, ALL records and other dirs untouched (atomic abort)
- [ ] Drifted with --force + confirmation → removed
- [ ] Unmanaged same-name dir in a non-recorded agent is never touched (fixture proves)
- [ ] Already-absent dir path: records cleaned, `(already absent)` note
- [ ] Unknown name → exit 3 with manifest-based suggestion

## Validation

`npm test -- remove` green on CI matrix.

## Dependencies

09, 10, 16, 17, 21, 22 (readLine only).

## Non-goals

Pruning orphan lock entries en masse (`install` warns; explicit removes only);
cache eviction (issue 32).

## Design References

- DESIGN.md 9 (remove), 10.3 (safety — normative), ADR-007
