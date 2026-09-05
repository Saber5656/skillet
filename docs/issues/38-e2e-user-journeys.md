# Title

End-to-end user-journey suite across the full command lifecycle

## Summary

Implement scenario tests that run the built CLI through complete real-world
journeys (team onboarding, personal setup, author loop, drift recovery) in
sandboxes with fixture registries — the regression net for v1 sign-off.

## Context

Unit and command tests verify parts; DESIGN 4 defines the journeys users
actually run. These tests execute `node dist/cli.js` (never internal imports)
so packaging, flag parsing, exit codes, and output contracts are exercised
end-to-end, matching ISSUE_PLAN's validation strategy.

## Scope

- `tests/e2e/*.test.ts`, journey helpers in `tests/e2e/journey.ts`
- CI: e2e job runs after build on the full OS matrix

## Detailed Requirements

Journeys (each a test file; steps assert stdout/stderr/exit/filesystem):

1. **team-project**: init → add registry skill (`--yes`) → verify manifest,
   lock, `.claude/skills`, `.codex/skills`, state → simulate teammate: copy
   `skillet.json`+`skillet.lock` (NOT `.skillet/`, NOT agent dirs) into a
   fresh sandbox project → `install --frozen --yes` → byte-identical skill
   trees (recursive compare) → `verify` exit 0.
2. **personal-global**: init -g → add -g two skills → list -g shows both →
   remove -g one → dirs and records consistent.
3. **author-loop**: scaffold local skill → `validate --print-integrity` →
   add `file:` → edit the local skill → `install` fails E_INTEGRITY →
   re-add → clean.
4. **update-flow**: registry fixture releases 1.1.0 while manifest has `^1.0.0`
   → `update --dry-run` shows table, no writes → `update --yes` → lock diff,
   re-materialized targets, `verify` clean.
5. **drift-recovery**: hand-edit an installed copy → `verify` exit 6 names it
   → `install` refuses to clobber? NO — install sees receipt-hash drift →
   E_TARGET_CONFLICT per issue 21 table → user follows the hinted
   `add --force` path → healthy again → `verify` exit 0. (This journey pins
   the exact recovery UX and its hint texts.)
6. **offline-flow**: warm cache → server down → `search`/`info` still answer
   (stale warning); `add` of an uncached package fails E_FETCH with the
   documented hint.
7. **json-contract**: every command run with `--json` across the journeys;
   stdout parses; envelope shape `{ok, command, data, error}` asserted by a
   shared helper on every invocation automatically.
8. Journey helper API: `journey(t).run(argv, {expectExit, env})` capturing
   trimmed streams, providing `expectFileTree`, `expectJson(path)` matchers;
   all journeys OS-portable (path assertions via helpers).

## Acceptance Criteria

- [ ] All 7 journeys green on ubuntu/macos/windows × Node 20/22
- [ ] Journey 1 proves byte-identical reproduction (the core product promise)
- [ ] No journey imports src/ modules directly (spawned CLI only; lint rule or test)
- [ ] Combined e2e wall time < 5 min per matrix cell
- [ ] Failures print full captured stdout/stderr for debuggability

## Validation

`npm test -- e2e` green on the full matrix; flake check: 3 consecutive CI runs
without retries.

## Dependencies

All command issues 18, 23-32 (journeys exercise them); 08 (sandbox), 36
(fixture patterns reusable).

## Non-goals

Performance benchmarking; real-network tests against github.com (never in CI).

## Design References

- DESIGN.md section 4 (journeys), 14 (contracts), 16 (testing strategy)
