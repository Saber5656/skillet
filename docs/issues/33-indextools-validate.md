# Title

indextools/validate.ts: index repository validation engine

## Summary

Implement the validation engine that index-repo CI (and maintainers locally)
run against registry entries: schema, naming/ownership rules, source
verification at pinned commits, integrity recomputation, and version
immutability.

## Context

DESIGN section 11 enumerates the checks; ADR-002/ADR-004/ADR-005 make version
immutability and owner binding security invariants. The engine reuses client
modules verbatim (schemas 05, skillmd 06, gitfetch 12/13, integrity 14,
stage filters 20) so registry and client can never disagree about validity.

## Scope

- `src/indextools/validate.ts`
- `tests/indextools/validate.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface IndexValidateOptions { repoDir: string;
     changedOnly: boolean;            // true in PR CI: validate entries whose files differ from `baseRef`
     baseRef: string | null;          // e.g. "origin/main"; null → treat all as changed
     network: boolean; }              // false → skip source verification (fast local lint)
   export interface Finding { entryFile: string; severity: "error" | "warning";
     rule: string; detail: string; }
   export function validateIndexRepo(env: Environment, o: IndexValidateOptions): Promise<{ findings: Finding[]; checkedEntries: number; }>;
   ```
2. Structural pass (always, all entries): every file under `registry/` matches
   `registry/<owner>/<skill>.json` (rule `layout`); parses as IndexEntry
   (rule `schema`); `name` == `<owner>/<skill>` from its path (rule
   `name-path-mismatch`); global case-insensitive uniqueness of names (rule
   `duplicate-name`); `source.url` https + host allowlist `github.com`,
   `gitlab.com`, `codeberg.org` (rule `source-host`); owner binding: URL owner
   segment equals entry owner case-insensitively (rule `owner-binding`);
   keywords/description bounds (schema covers; re-report as findings not
   throws); semver validity + strictly-increasing ordering within the file
   (rule `version-order`).
3. Immutability pass (requires `baseRef`): read the same entry file at
   `baseRef` via `git show <baseRef>:<file>` in `repoDir`; every version
   present in base MUST appear in head with identical `commit`, `path`, AND
   `integrity` (rule `version-immutable`); version removals allowed only when
   the head entry marks `deprecated` non-null (rule `version-removed`).
4. Source pass (network, changed entries only): for each NEW/CHANGED version:
   `ensureCommit` → `listTree(commit, path)` → `validateTreeEntries` (all
   violations become findings, rule prefixed `tree:`) → extract to stage →
   integrity recompute == declared (rule `integrity-mismatch`) →
   `inspectSkillDir`: SKILL.md valid, `meta.name` == skill segment (rule
   `skillmd`), description in entry == SKILL.md description verbatim (rule
   `description-sync`), `metadata.version` if present == version (rule
   `metadata-version`).
5. `changedOnly` file set from `git diff --name-only <baseRef>...HEAD --
   registry/` in repoDir.
6. Determinism: findings sorted by (entryFile, rule, detail); the engine never
   throws for content problems (only for E_CONFIG-class environment issues) —
   CI wants the full report.

## Acceptance Criteria

- [ ] Fixture index repo (built with issue-08 git helpers, entries pointing at fixture source repos): clean repo → zero findings
- [ ] One fixture per rule id above producing exactly that finding (≥ 12 rules)
- [ ] Immutability: changing a published version's commit → `version-immutable`; adding a version → clean
- [ ] `network: false` skips the source pass but still catches structural issues
- [ ] changedOnly validates only touched entries (spy on ensureCommit count)

## Validation

`npm test -- indextools-validate` green (POSIX CI; Windows included).

## Dependencies

05, 06, 08, 12, 13, 14, 20.

## Non-goals

Aggregate building (issue 34); CLI wiring (issue 35); homoglyph/unicode
confusable detection (v2, DESIGN 15).

## Design References

- DESIGN.md section 11 (checks — normative), 7.5 (entry), ADR-002/004/005
