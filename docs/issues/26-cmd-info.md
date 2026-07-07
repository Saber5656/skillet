# Title

skillet info: show package metadata, versions, and deprecation

## Summary

Implement `skillet info <package-id>[@range]` printing the index record in a
readable block plus machine JSON.

## Context

DESIGN 14 command table. Discovery workflow step between search and add
(DESIGN 4 workflow 3). Range argument filters the version list.

## Scope

- `src/cli/commands/info.ts`
- `tests/cli/info.test.ts`

## Detailed Requirements

1. Synopsis: `skillet info <id>[@range]`. Id validated via `parsePackageId`
   (E_USAGE); unknown → E_UNKNOWN_PACKAGE with nearest-name hints (client,
   issue 11).
2. Human output block:
   ```
   anthropics/pdf — Extract PDF text and tables, fill forms, merge files.
     source:   https://github.com/anthropics/skills (path: skills/pdf @ latest)
     license:  MIT    keywords: pdf, documents, forms
     maintainers: anthropics
     versions: 1.2.0 (2026-07-01), 1.1.0 (2026-05-12), 1.0.0 (2026-04-02)
   ```
   Deprecated → prominent yellow block with reason + replacement. With
   `@range`: versions list filtered to satisfying versions; none → 
   E_NO_MATCHING_VERSION listing available.
3. Versions sorted desc; show ≤ 10 then `… N older versions`.
4. `--json` data: the full IndexEntry plus `{ matchingVersions: […] }` when a
   range was given.
5. Offline behavior identical to search (offlineOk + staleness warning).

## Acceptance Criteria

- [ ] Snapshot test of the human block (fixture entry with 12 versions → truncation line)
- [ ] `@^1` filters correctly; `@^9` → exit 3 listing available versions
- [ ] Deprecated rendering includes reason and replacement id
- [ ] Unknown id suggests `anthropics/pdf` for input `anthropic/pdf`
- [ ] `--json` returns the untruncated entry

## Validation

`npm test -- info` green.

## Dependencies

03, 04, 07, 11, 17.

## Non-goals

Fetching README/SKILL.md bodies from sources (v2 `info --readme`).

## Design References

- DESIGN.md 14 (command), 7.5 (entry fields), 4 (workflow 3)
