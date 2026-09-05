# Title

skillet index validate|build: CLI wiring for index tooling

## Summary

Expose the indextools engines as `skillet index validate [dir]` and
`skillet index build [dir] [--check]` with CI-friendly flags and outputs, so
the index repository's workflows are one-liners.

## Context

DESIGN 14 command table and section 11: the index repo CI runs
`npx @skillet/cli index validate --changed-since origin/main` on PRs and
`skillet index build` on main. Findings must render readably in GitHub Actions
logs and as JSON for tooling.

## Scope

- `src/cli/commands/index.ts`
- `tests/cli/index-cmd.test.ts`

## Detailed Requirements

1. `skillet index validate [dir]` flags: `--changed-since <ref>` (sets
   changedOnly+baseRef), `--no-network` (structural only), `--strict`
   (warnings count as failures). Dir default `.` and must contain `registry/`
   (E_USAGE otherwise with a hint that this runs in the index repo, not skill
   repos).
2. Output (human): findings grouped by entry file with `error:`/`warning:`
   prefixes and rule ids; GitHub Actions annotations
   (`::error file=<f>::<rule>: <detail>`) emitted when `GITHUB_ACTIONS=true`;
   summary `N errors, M warnings across K entries`.
3. Exit codes: 0 clean (or warnings without `--strict`); 4 any error finding
   (E_SPEC_INVALID class); 3 for environment failures (network, git).
4. `skillet index build [dir] [--check]`: builds via issue 34; default writes
   `index.json` and prints its byte size + package count; `--check` writes
   nothing, exit 0 up-to-date / exit 4 stale with the diff summary.
5. `--json` for both: validate → `{ findings, checkedEntries, errors, warnings }`;
   build → `{ packages, bytes, wrote | upToDate }`.
6. `generatedAt` for build: from `git log -1 --format=%cI` in dir (issue 34
   rule); repo without commits → E_USAGE (index repo is always a git repo).

## Acceptance Criteria

- [ ] Fixture index repo: validate clean → exit 0; injected bad entry → exit 4 with annotation line when GITHUB_ACTIONS=1
- [ ] `--changed-since` validates only the touched entry (spy)
- [ ] `--no-network` never spawns git fetch (spy on run())
- [ ] build writes deterministic aggregate; `--check` catches staleness; timestamp-only diff passes
- [ ] `--strict` turns a warnings-only run into exit 4

## Validation

`npm test -- index-cmd` green.

## Dependencies

17, 33, 34.

## Non-goals

The index repository content itself (issue 36); scheduled re-validation of old
entries (v2).

## Design References

- DESIGN.md 14 (command), section 11 (CI usage)
