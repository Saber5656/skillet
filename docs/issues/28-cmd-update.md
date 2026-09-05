# Title

skillet update: range-constrained upgrades with lock diff table

## Summary

Implement `skillet update [names…]`: refresh the index, re-resolve within
manifest ranges (all or named packages), show the old→new table, update the
lock, and re-materialize changed packages.

## Context

DESIGN 9 update row is normative: ranges come from the manifest (changing a
range = re-add); direct-git branch refs re-resolve only when explicitly named;
unchanged resolutions are `noop`. Consent: a changed integrity is a new trust
decision (issue 22 `needsConsent` handles it naturally).

## Scope

- `src/cli/commands/update.ts`
- `tests/cli/update.test.ts`

## Detailed Requirements

1. Synopsis: `skillet update [names…] [--scope|-g] [--agents ids…] [--yes]
   [--dry-run]`.
2. Index fetched with `refresh: true` (revalidation forced — update is the
   command that must see new releases).
3. `planUpdate` (issue 19); names validated first (unknown → exit 3 before any
   staging).
4. `--dry-run`: print the change table and exit 0 without staging, recording,
   or materializing (CI-friendly "outdated" check: rows present → print table;
   also exit 0 — outdatedness is not an error; document that scripts should
   use `--json` and inspect `changes.length`).
5. Change table (human): columns `PACKAGE  FROM  TO  COMMIT` (short SHAs);
   noop rows omitted; `all up to date` when empty. `--json` data:
   `{ changes: [{ id, fromVersion, toVersion, fromCommit, toCommit }],
   updated: […], dryRun }`.
6. Execution order per changed package = add pipeline stages FETCH→…→
   MATERIALIZE with lock-entry replacement (RECORD once per package, same
   rollback contract as issue 23 step 7); `diffLockfiles` (issue 10) renders
   the final summary.
7. Deprecation discovered during update (package now deprecated in index) →
   warning + consent requirement even if version unchanged? NO — noop rows
   never prompt; deprecation warning printed once (informational). Only
   version/integrity changes prompt. Test pins this.

## Acceptance Criteria

- [ ] Fixture index v1.2.0→v1.3.0: update resolves, prompts (integrity changed), lock diff rendered, both agents re-materialized
- [ ] Range `~1.2.0` does NOT move to 1.3.0 (respect ranges)
- [ ] `update <name>` touches only that package; unnamed direct-git branch dep untouched; named one re-resolves its branch head
- [ ] `--dry-run` writes nothing (manifest/lock/state/dirs byte-identical)
- [ ] `all up to date` path exits 0 quickly (no staging calls — spy)
- [ ] Declined consent mid-update leaves that package at the old version, lock consistent

## Validation

`npm test -- update` green on CI matrix.

## Dependencies

10, 11, 12, 17, 19, 20, 21, 22, 23 (shared pipeline helpers extracted there).

## Non-goals

Range-widening upgrades (`add` handles); bulk `--latest` overrides (v2);
changelog display (v2).

## Design References

- DESIGN.md 9 (update — normative), 10.1, 12.4
