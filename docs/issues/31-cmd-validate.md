# Title

skillet validate: author-facing skill directory validation

## Summary

Implement `skillet validate [path]`: run the exact spec + safety + limits
checks against a local skill directory and print an actionable report,
including the integrity string authors need for index PRs.

## Context

DESIGN 14 command table and ADR-004: authors validate locally with the same
code index CI runs (issues 06/20/14), then copy `--print-integrity` output
into their `registry/<owner>/<name>.json` PR. This closes the author loop
without a publish command.

## Scope

- `src/cli/commands/validate.ts`
- `tests/cli/validate.test.ts`

## Detailed Requirements

1. Synopsis: `skillet validate [path] [--print-integrity]` (path default `.`;
   must contain `SKILL.md` — otherwise E_USAGE with hint when a `skills/`
   subdir with candidates exists: list them).
2. Checks, in order, ALL executed (report accumulates; no fail-fast):
   1. Spec validation via `inspectSkillDir` (issue 06) — errors + warnings.
   2. Safety/limits via `stageFromLocalPath`'s filter path (issue 20) —
      reuse `validateTreeEntries` on synthesized entries to get the full
      violation list with rule ids.
   3. Advisory checks (warnings only): body > 48 KiB (spec guidance
      "keep SKILL.md under 500 lines"); `metadata.version` present and valid
      semver (recommended for registry publishing); LICENSE file present when
      frontmatter `license` names one.
3. Output (human): `✓`/`✗`/`!` lines per check group, then
   `valid: <name> (N files, X KiB, M executables)` or an error summary; with
   `--print-integrity`: final line exactly `integrity: sha256-…` (stdout, also
   present in `--json` data) computed via `hashLocalDir`.
4. Exit codes: 0 valid (warnings allowed); 4 (`E_SPEC_INVALID`/`E_UNSAFE_TREE`
   class) when any error-severity finding exists.
5. `--json`: `{ valid, name, fileCount, totalBytes, executables, integrity?,
   errors: [{rule, path, detail}], warnings: […] }`.
6. This command must not read manifest/lock/state (pure author tool, works
   outside any skillet project).

## Acceptance Criteria

- [ ] Valid fixture skill → exit 0; with `--print-integrity` the string equals issue-14 vector recomputation
- [ ] name/dir mismatch → exit 4 with both names in message
- [ ] Symlink inside → exit 4 with rule id `symlink`
- [ ] Multi-error fixture reports ALL findings in one run
- [ ] Advisory-only fixture (huge body, missing metadata.version) → exit 0 with 2 warnings
- [ ] Works in a directory with no skillet.json anywhere

## Validation

`npm test -- validate` green.

## Dependencies

06, 14, 17, 20.

## Non-goals

Registry entry JSON generation (v2 `skillet publish` per ADR-004); network.

## Design References

- DESIGN.md 14 (command), section 8 (limits), ADR-004 (author workflow)
