# Title

core/materialize.ts: atomic install/replace/remove in agent skills directories

## Summary

Implement the only module that writes into runtime directories: atomic
sibling-rename installs, managed-only replacement with hash verification,
managed-only removal, and rollback on partial failure.

## Context

DESIGN.md 10.3 and ADR-007 are normative: copy-based, receipts required for
any destructive action, `E_TARGET_CONFLICT` on unmanaged/drifted targets,
`--force` relaxes only the hash check. Multi-target installs (N agents) must
either fully succeed per target or roll that target back.

## Scope

- `src/core/materialize.ts`
- `tests/core/materialize.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface Target { agent: string; skillsDir: string; dir: string; }   // dir = join(skillsDir, skillName)
   export function planTargets(agents: Adapter[], scope: "user"|"project",
     roots: { home: string; projectRoot: string | null }, skillName: string): Target[];
   export interface MaterializeResult { target: Target; outcome: "installed" | "replaced" | "unchanged"; }
   export function materialize(staged: StagedSkill, target: Target, state: State,
     opts: { force: boolean }): Promise<{ result: MaterializeResult; receipt: InstallReceipt }>;
   export function unmaterialize(target: Target, state: State,
     opts: { force: boolean }): Promise<void>;   // E_TARGET_CONFLICT / no-op if absent with receipt cleanup
   ```
2. `materialize` decision table (target dir existence × receipt × hash):
   | exists | receipt | hash matches receipt | action |
   |---|---|---|---|
   | no | any | — | fresh install (copy → tmp sibling → rename) |
   | yes | none | — | `E_TARGET_CONFLICT` ("unmanaged directory", hint: remove/rename or choose another skill name) — `--force` does NOT override |
   | yes | yes | yes | if receipt.integrity == staged.integrity → outcome `unchanged` (no writes); else swap-replace |
   | yes | yes | no (drifted) | `E_TARGET_CONFLICT` ("locally modified", hint: `skillet verify`, re-add with `--force` to overwrite) — `--force` DOES override → swap-replace |
3. Swap-replace protocol: copy staged → `<dir>.skillet-new-<rand>`; rename
   `<dir>` → `<dir>.skillet-old-<rand>`; rename new → `<dir>`; delete old.
   Failure between the two renames restores old (rename back) before
   rethrowing `E_MATERIALIZE`. All temp names excluded from receipts.
4. Copies preserve the exec bit on POSIX (`fs.cp` with manual chmod pass from
   staged meta.executables); Windows: plain copy.
5. Fresh install into a missing `skillsDir` creates it (`mkdir -p`); creating
   the ADAPTER BASE dir (e.g. `~/.codex`) is allowed for user scope only when
   the manifest/flags explicitly named that agent (i.e., not via detection —
   detection never invents a runtime); pass a `explicitlyRequested: boolean`
   on Target for this rule (planTargets sets it from selection source, issue 15).
6. `unmaterialize`: no receipt → `E_TARGET_CONFLICT` (never delete unmanaged);
   receipt + hash match → delete dir; drifted → `E_TARGET_CONFLICT` unless
   `force`; dir already absent → succeed (receipt removal happens in caller
   via state module). Deletion uses `fs.rm(dir, { recursive: true })` only
   after a final `receiptForDir` re-check (belt and braces).
7. The module returns receipts but never writes state.json (caller owns state
   writes AFTER all targets processed — crash between materialize and state
   write must err on the side of "receipt missing" which blocks future
   destructive ops; document this ordering in a comment).

## Acceptance Criteria

- [ ] Decision table: all 5 rows covered by tests, incl. both `--force` behaviors
- [ ] Rollback: injected failure after old-rename restores the original dir bit-for-bit
- [ ] `unchanged` outcome does zero writes (mtime assertions)
- [ ] Unmanaged same-name dir survives every code path
- [ ] Exec bit preserved (POSIX assertion; skipped on Windows with reason)
- [ ] Detection-selected codex without `~/.codex` → target skipped with warning, not created (explicitlyRequested=false path)

## Validation

`npm test -- materialize` green on CI matrix.

## Dependencies

02, 03, 15, 16, 20.

## Non-goals

State persistence ordering (command issues); consent (issue 22); symlink
strategies (rejected by ADR-007).

## Design References

- DESIGN.md 10.3 (normative), 13 (targets), ADR-007
