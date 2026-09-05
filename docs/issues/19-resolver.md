# Title

core/resolver.ts: range resolution and manifest/lock reconciliation planning

## Summary

Implement the pure planning core that turns (manifest, lockfile, index,
intent) into an executable per-package plan — the single place where DESIGN
section 9 semantics live.

## Context

DESIGN.md section 9 fixes the semantics for add/install/frozen/update/remove,
conflict rules (`E_NAME_CONFLICT`, `E_UNKNOWN_PACKAGE`,
`E_NO_MATCHING_VERSION`), deprecation handling, and pseudo-versions for
direct-git packages. Commands (issues 23/24/28/29) are thin shells over this
module plus fetch/stage/materialize.

## Scope

- `src/core/resolver.ts`
- `tests/core/resolver.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface ResolveInput { manifest: Manifest; lock: Lockfile | null;
     index: AggregateIndex | null;    // null allowed when no registry deps exist
     refResolver: (kind: "github" | "git", url: string, ref: string | null) => Promise<string>; // injected (issue 12); resolver stays pure/testable
   }
   export interface PlannedPackage { id: string; skillName: string;
     action: "install" | "update" | "reinstall" | "noop" | "remove";
     from?: LockedSkill; to?: LockedSkill;      // `to` absent for remove
     deprecated?: { reason: string; replacement: string | null }; }
   export interface Plan { packages: PlannedPackage[]; lockChanged: boolean; }

   export function planInstall(i: ResolveInput, opts: { frozen: boolean }): Promise<Plan>;
   export function planAdd(i: ResolveInput, adds: Array<{ ref: AddRef; }>): Promise<Plan>;
   export function planUpdate(i: ResolveInput, names: string[] | null): Promise<Plan>;
   export function planRemove(i: ResolveInput, names: string[]): Promise<Plan>;
   ```
2. Registry resolution: candidate versions from the index entry, filtered by
   `semver.satisfies`, sorted desc by `semver.compare`; `"latest"` = highest
   non-prerelease; prereleases only match when the range explicitly includes
   them (semver default `includePrerelease: false`). No match →
   `E_NO_MATCHING_VERSION` listing up to 10 available versions. Unknown id →
   `E_UNKNOWN_PACKAGE` with `nearestNames` hints (issue 11).
3. `planInstall` (lock present): every manifest entry with a satisfying lock
   entry → action from state-agnostic view is `install` (materialization may
   later no-op); manifest entry missing from lock → resolve fresh
   (`lockChanged: true`); lock entries absent from manifest → `remove`.
   `frozen: true` → any of {missing lock, missing entry, unsatisfied range,
   orphan lock entry, `portable: false` entry} → `E_FROZEN_DRIFT` whose message
   enumerates every violation (not just the first).
4. `planAdd`: parse each AddRef; registry refs resolve like above and the
   manifest range defaults to `^<version>` (DESIGN 9) unless the user gave an
   explicit range (preserved verbatim). Direct refs call `refResolver` to get
   the SHA now; id becomes known only post-staging, so `planAdd` returns
   direct entries with `id: ""` and a `pendingIdentity: true` marker field —
   the add command (issue 23) finalizes id after staging and re-checks
   collisions via `assertNoNameConflicts` exported from this module.
5. `planUpdate`: names null → all registry entries; named direct-git entries
   with branch refs re-resolve via refResolver; named entries not in manifest →
   `E_UNKNOWN_PACKAGE`. Result rows where version/commit unchanged → `noop`.
6. Conflict pass (all plans): duplicate skill-name basenames across final
   entries → `E_NAME_CONFLICT` naming both package ids.
7. Deprecation: index entry deprecated → attach to planned package; plans never
   fail on deprecation (consent layer decides, DESIGN 9).
8. Pure module: no fs, no network, no reporter; deterministic given inputs.

## Acceptance Criteria

- [ ] Range matrix: `^1.2.0`, `~1.2.0`, `1.2.3`, `latest`, prerelease range `^2.0.0-rc` against a 7-version fixture — expected picks asserted
- [ ] frozen violations: each of the five cases + a combined case listing all in one message
- [ ] add default-range rule (`^1.4.2`) and explicit-range preservation
- [ ] update: noop rows for unchanged; direct branch re-resolve only when named
- [ ] name-conflict: `anthropics/pdf` + `acme/pdf` → E_NAME_CONFLICT both ids in message
- [ ] unknown package suggests nearest names

## Validation

`npm test -- resolver` green; 100% branch coverage on this module
(`vitest --coverage` threshold set in the test file's describe block docs).

## Dependencies

02, 05, 07, 10, 11 (types + nearestNames), 12 (refResolver signature only).

## Non-goals

Fetching, staging, consent, materialization (issues 12/20/22/21); CLI wiring
(issues 23/24/28/29).

## Design References

- DESIGN.md section 9 (normative semantics), 6.2 (pseudo-versions), 15 (deprecation row)
