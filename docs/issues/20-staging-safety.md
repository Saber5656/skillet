# Title

core/stage.ts: staging pipeline with safety filters and limits

## Summary

Implement the FETCH→EXTRACT→VERIFY composition: create stage directories,
apply the full safety filter + limits to enumerated trees, extract, hash,
spec-validate, and hand back a verified `StagedSkill` ready for consent and
materialization.

## Context

DESIGN.md 10.1 (EXTRACT/VERIFY phases), section 8 (filters and limits:
1000 files, 20 MiB total, 2 MiB/file, depth 16, path ≤ 512 bytes; reject
symlinks/gitlinks/case-fold dupes), 12.2 (threat rows). This module is shared
verbatim by index CI validation (issue 33) so client and registry agree on
what "safe" means.

## Scope

- `src/core/stage.ts`
- `tests/core/stage.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface TreeViolation { relPath: string | null; rule: string; detail: string; }
   export function validateTreeEntries(entries: TreeEntry[]): TreeViolation[];  // returns ALL violations (index CI needs the full list)
   export function assertSafeTree(entries: TreeEntry[]): void;                  // throws E_UNSAFE_TREE with first 5 violations summarized

   export interface StagedSkill { dir: string; integrity: string; meta: SkillDirInfo;
     source: ResolvedSource; cleanup(): Promise<void>; }
   export function stageFromGit(env: Environment, url: string, commit: string, path: string,
     expectedIntegrity: string | null): Promise<StagedSkill>;
   export function stageFromLocalPath(env: Environment, absPath: string): Promise<StagedSkill>;
   ```
2. Filter rules (`validateTreeEntries`) — rule ids fixed for tests and CI
   output: `symlink`, `gitlink`, `non-blob`, `bad-mode`, `traversal`
   (relPath rules incl. `..`, `\`, NUL, leading `/`), `case-collision`,
   `too-many-files` (>1000), `file-too-large` (>2 MiB), `tree-too-large`
   (>20 MiB total), `too-deep` (>16 segments), `path-too-long` (>512 bytes),
   `empty-tree` (0 blobs), `missing-skillmd` (no root `SKILL.md`).
3. `stageFromGit` sequence: `ensureCommit` → `listTree` → `assertSafeTree` →
   stage dir `<cacheDir>/stage/<random>` → `extractTree` → integrity :=
   `hashExtractedEntries`; if `expectedIntegrity` non-null and different →
   E_INTEGRITY (message shows both values and names the threat: "content does
   not match the registry/lockfile record") and stage dir removed → then
   `inspectSkillDir` (spec validation; failure E_SPEC_INVALID also cleans up).
4. `stageFromLocalPath`: enumerate local dir with lstat (symlink → violation),
   synthesize TreeEntry-equivalents (cls from exec bit), run the same
   `assertSafeTree`, copy to stage dir, hash with `hashLocalDir`.
5. `cleanup()` idempotent; stage dirs also removed on any thrown error inside
   the functions (try/finally); orphaned stage dirs are handled by
   `cache clean` (issue 32).
6. Concurrency: stage dir names use `crypto.randomUUID()`; no shared mutable
   state.

## Acceptance Criteria

- [ ] Each filter rule has a dedicated fixture producing exactly that rule id (13 rules × ≥ 1 case); multi-violation fixture returns the full list from `validateTreeEntries` while `assertSafeTree` throws once
- [ ] expectedIntegrity mismatch → E_INTEGRITY, stage dir gone (fs assertion)
- [ ] Happy path returns meta incl. executables + integrity equal to issue-14 recomputation
- [ ] Local path with symlink → E_UNSAFE_TREE
- [ ] Error paths never leave a stage dir behind

## Validation

`npm test -- stage` green on CI matrix.

## Dependencies

02, 03, 06, 08, 12, 13, 14.

## Non-goals

Consent (issue 22); materialization (issue 21); limits configurability (fixed
in v1).

## Design References

- DESIGN.md section 8 (limits/filters — normative), 10.1, 12.2; ADR-005, ADR-008
