# Title

skillet search: query the index from the cache-backed client

## Summary

Implement `skillet search <query> [--keyword k]…` rendering ranked matches as
a table (human) or structured JSON.

## Context

DESIGN 14 command table; ranking is already implemented and tested in the
index client (issue 11) — this issue is presentation + flag plumbing only.
Search must work offline from a warm cache (DESIGN 7.6).

## Scope

- `src/cli/commands/search.ts`
- `tests/cli/search.test.ts`

## Detailed Requirements

1. Synopsis: `skillet search <query> [--keyword <k>]… [--limit n]` (limit
   default 20, max 100). Empty query with no keywords → E_USAGE.
2. Fetch index with `offlineOk: true`; print the staleness warning from the
   client when `stale` (`warning: index cache is N days old; run any command
   online to refresh`).
3. Human output: table `NAME  VERSION  DESCRIPTION` where VERSION is the
   highest non-deprecated release, DESCRIPTION truncated to terminal width
   (min 40 cols, single line, ellipsis); deprecated entries suffixed
   `(deprecated)` in yellow. Zero hits → `no packages matched "<query>"` on
   stderr, exit 0 (not an error).
4. `--json` data: `{ query, keywords, total, hits: [{ name, description,
   latestVersion, deprecated, score, keywords }] }` (post-limit `hits`,
   pre-limit `total`).
5. No index mutation; no consent; read-only command.

## Acceptance Criteria

- [ ] Fixture index (6 packages from issue 11's ranking test): ordering matches the client ranking; limit respected
- [ ] `--keyword` all-must-match behavior surfaces client semantics
- [ ] Offline warm cache works; cold cache → exit 3 with connectivity hint
- [ ] Zero hits → exit 0 with the documented message
- [ ] `--json` matches the schema; stdout parses; human text only on stderr

## Validation

`npm test -- search` green.

## Dependencies

03, 04, 11, 17.

## Non-goals

Fuzzy/interactive search (v2); popularity signals (v2).

## Design References

- DESIGN.md 14 (command), 7.6 (cache/offline), 11 issue (ranking contract)
