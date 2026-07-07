# Title

skillet install: reproduce the lockfile, including --frozen CI mode

## Summary

Implement `skillet install [--frozen]`: materialize exactly what the lockfile
records for every manifest entry (resolving only lock-missing entries), the
team/CI reproducibility workflow.

## Context

DESIGN 9 install/frozen rows; DESIGN 4 workflow 1 (teammate/CI). Consent
nuance: installing from a lockfile that the user's repo already contains is
still a first materialization on THIS machine → consent applies per DESIGN
12.4, and `--frozen` is typically paired with `--yes` in CI (document in help
text).

## Scope

- `src/cli/commands/install.ts`
- `tests/cli/install.test.ts`

## Detailed Requirements

1. Synopsis: `skillet install [--frozen] [--scope|-g] [--agents ids…] [--yes]`.
2. `planInstall` (issue 19). Frozen violations → E_FROZEN_DRIFT (exit 3) with
   the full violation list, before any network/staging.
3. Non-frozen with lock-missing entries: fetch index (only if needed), resolve,
   and write the updated lock after successful staging of those entries
   (RECORD before MATERIALIZE, mirroring issue 23 ordering and rollback).
4. Staging uses lock integrity as `expectedIntegrity` for `index`/`git`
   entries (tamper detection on every reinstall); `path` entries re-hash and
   REFUSE with E_INTEGRITY if the local dir hash differs from lock (hint:
   re-run `skillet add file:…` to accept changes) — local drift must be
   explicit, never silent.
5. Per-package × per-target materialization with the issue-21 `unchanged`
   fast path; a fully `unchanged` run prints `up to date (N skills, M targets)`
   and exits 0 without consent prompts (needsConsent false).
6. `remove`-action rows from the plan (orphan lock entries in non-frozen mode)
   are NOT executed by install; instead print a warning listing them with hint
   `skillet remove <id>` (install never deletes — conservative).
7. Exit codes: success 0 even when some targets were `unchanged`; first
   hard error aborts the run after attempting rollback of the in-flight
   package only (already-completed packages stay — install is resumable by
   re-running; document).
8. `--json` data: `{ installed: […], unchanged: […], resolvedNew: […],
   orphanLockEntries: […] }`.

## Acceptance Criteria

- [ ] Clean checkout simulation: manifest+lock present, empty agent dirs → all skills materialized byte-identical to staged content; second run prints `up to date` with zero writes
- [ ] `--frozen` with: no lock / missing entry / range-unsatisfied lock / orphan entry / portable:false — each exits 3 naming the violation; combined fixture lists all
- [ ] Lock-missing entry (non-frozen): resolved, lock updated, materialized
- [ ] Tampered source (fixture repo rewrites the tag's content → different tree at same recorded commit is impossible; simulate via wrong integrity in lock) → exit 4, nothing written
- [ ] path entry drifted → exit 4 with the documented hint
- [ ] Orphan lock entry warning path (no deletion)

## Validation

`npm test -- install` green; e2e: run in a second sandbox "machine" cloning
the first sandbox's project dir (fixture) to prove reproducibility.

## Dependencies

09, 10, 11, 12, 15, 16, 17, 19, 20, 21, 22.

## Non-goals

update semantics (issue 28); removing orphans (issue 29); workspace/monorepo
multi-manifest (v2).

## Design References

- DESIGN.md 9 (install/frozen — normative), 4 (workflow 1), 10.1, 12.4
