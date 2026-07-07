# Title

skillet add: resolve, stage, consent, record, materialize new packages

## Summary

Implement `skillet add <ref…>` end-to-end by composing resolver, stage,
consent, manifest/lock recording, and materialization in the exact DESIGN 10.1
phase order with rollback of recorded files if materialization fails.

## Context

This is the flagship write path (DESIGN 9 add row; 10.1 state machine).
Registry refs default to `^<resolved>` ranges; direct git refs pin SHAs at add
time; local paths are dev-only (`portable: false`). Direct-git identity is
known only after staging (issue 19's `pendingIdentity` contract).

## Scope

- `src/cli/commands/add.ts`
- `tests/cli/add.test.ts` (sandboxed e2e through the built CLI)

## Detailed Requirements

1. Synopsis: `skillet add <ref…> [--scope|-g] [--agents ids…] [--yes]`.
   At least one ref (E_USAGE otherwise). Refs parsed via `parseAddRef`.
2. Pipeline per DESIGN 10.1, for the batch:
   1. Load env/scope roots; manifest required (hint: `skillet init`) —
      E_CONFIG if absent. Lock loaded if present; index fetched only when a
      registry ref exists in the batch.
   2. `planAdd` (issue 19).
   3. For each planned package sequentially (deterministic order: input
      order): stage (`stageFromGit`/`stageFromLocalPath` with
      expectedIntegrity from index when registry-sourced, else null).
   4. Direct refs: finalize identity (owner from source per DESIGN 6.2, skill
      from staged meta.name); re-run `assertNoNameConflicts` against manifest
      + earlier batch items; version = `0.0.0-git.<shortsha12>` or
      `0.0.0-local`.
   5. Consent (issue 22) once per package with full summary incl. targets.
   6. RECORD: upsert manifest + lock entries; write both atomically (manifest
      first, then lock; on lock-write failure restore previous manifest bytes).
   7. MATERIALIZE to every selected target (issue 21); collect receipts; write
      state once at the end. If ANY target fails: roll back that package's
      manifest+lock entries to pre-add bytes, attempt unmaterialize of
      already-written targets for that package (receipts in memory), keep
      earlier fully-committed packages of the batch intact, and exit with the
      materialization error (partial-batch semantics documented in help text).
   8. Cleanup stage dirs (always).
3. Consent declined for one package aborts ONLY that package (exit code 5 if
   it was the only ref; with multiple refs: skip it, continue, final exit code
   5 if any declined — precedence: highest severity among {materialize error,
   declined} — document: error > declined > success).
4. Human output per package: one status line
   `+ anthropics/pdf@1.2.0 → claude-code (project), codex (project)`; final
   summary `N added, M skipped`. `--json` data:
   `{ added: [{id, version, integrity, targets: […]}], skipped: [{ref, reason}] }`.
5. Deprecated packages: warning + consent path (issue 22) even with `--yes`?
   NO — `--yes` covers deprecation too (single auto-approve rule; DESIGN 9's
   "requires --yes or interactive confirm"). Test both.
6. Re-adding an existing id with a new range: allowed; treated as range edit +
   re-resolve (this is the documented way to change ranges, DESIGN 9 update
   note).

## Acceptance Criteria

- [ ] Registry add happy path: manifest gains `^1.2.0`-style range; lock pinned; both agents' dirs contain the skill; state has 2 receipts; exit 0
- [ ] `github:` add: manifest stores SHA-pinned specifier string; version pseudo `0.0.0-git.<sha12>`
- [ ] `file:` add: lock `portable: false`; warning printed
- [ ] Integrity mismatch vs index → exit 4, nothing recorded, no target written
- [ ] Non-TTY without --yes → exit 5, nothing recorded
- [ ] Materialize conflict (pre-existing unmanaged dir) → exit 4, manifest/lock byte-identical to before (fixture compare)
- [ ] Multi-ref batch with one declined: other package fully installed; exit 5
- [ ] Name conflict across batch refs → exit 3, nothing written

## Validation

`npm test -- add` green on CI matrix; manual smoke against a local fixture
repo via `SKILLET_ALLOW_FILE_GIT=1`.

## Dependencies

07, 09, 10, 11, 12, 15, 16, 17, 19, 20, 21, 22.

## Non-goals

`install`/`update`/`remove` (issues 24/28/29); parallel staging (v2).

## Design References

- DESIGN.md 9 (add), 10.1 (phases + rollback), 12.4 (consent), 6.2 (identity)
