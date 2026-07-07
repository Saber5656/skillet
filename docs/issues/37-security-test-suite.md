# Title

Security test suite: adversarial fixtures proving every threat-table mitigation

## Summary

Build a dedicated test suite of hostile fixtures — malicious trees, injection
attempts, tampered content, conflict attacks — asserting that each row of the
DESIGN threat table fails closed with the specified error, so security
regressions become red CI.

## Context

DESIGN 12.2 lists mitigations; issues 12/13/20/21 implement them with unit
tests. This issue adds the adversarial END-TO-END layer: attacks executed
through the real CLI against sandboxes, because composition bugs (a filter
bypassed by a different code path) are where installers get owned.

## Scope

- `tests/security/*.test.ts` (new vitest project/tag `security`)
- `tests/fixtures/hostile/` builders (extending issue-08 gitfixture)
- CI: security tests run in the standard matrix (no special job needed)

## Detailed Requirements

Each scenario = hostile fixture + CLI invocation + assertions on exit code,
error code in `--json`, and full filesystem non-effect (agent dirs, manifest,
lock, state byte-compared before/after):

1. **Symlink escape**: repo with `SKILL.md` + symlink `evil -> ../../../…`;
   `add github:…` → exit 4 `E_UNSAFE_TREE`; no file created outside sandbox
   stage (assert via walk).
2. **Gitlink/submodule**: same shape → `E_UNSAFE_TREE`.
3. **Traversal names**: paths with `..` and backslash entries created via git
   plumbing (`git update-index --index-info`) → `E_UNSAFE_TREE`.
4. **Case-collision smuggle**: `README.md`/`readme.md` → `E_UNSAFE_TREE`
   (POSIX runner).
5. **Oversize attacks**: 1001 files; single 3 MiB file; 21 MiB total → each
   rejected with its rule id.
6. **Integrity tamper**: lockfile integrity altered by one char → install →
   exit 4 `E_INTEGRITY`, no materialization.
7. **Index tamper**: fixture index serves an entry whose integrity does not
   match its actual tree → add → exit 4, nothing recorded.
8. **Ref injection**: `skillet add "github:a/b#--upload-pack=touch /tmp/pwn"`
   → exit 2 `E_USAGE` before any git spawn (spy) — plus scp-style URL with a
   leading-dash host rejected.
9. **Unmanaged clobber**: pre-create victim skill dir (user-authored) with the
   same name → add → exit 4 `E_TARGET_CONFLICT`, victim bytes untouched;
   remove of the same name (no receipt) → refuses.
10. **Consent bypass probe**: non-TTY add without `--yes` → exit 5 and zero
    disk/record effects (already covered in 23 — re-asserted here as the
    composition invariant).
11. **Hostile index URL via project files**: a fixture project containing
    `skillet.json` with an extra `indexUrl` key → key ignored by schema
    (strip), requests still go to the env-configured URL (server request-log
    assertion) — proves DESIGN 12.3.
12. **YAML bomb SKILL.md**: alias-expansion payload in a fixture repo →
    `E_SPEC_INVALID` fast (< 5 s guard).
Each scenario name cites its threat-table row in a comment; a meta-test
asserts every 12.2 row id has ≥ 1 scenario (keep a mapping table in
`tests/security/coverage.ts`).

## Acceptance Criteria

- [ ] All 12 scenario groups implemented and green on the CI matrix (case-collision POSIX-only with skip reason)
- [ ] Every scenario asserts BOTH the error code (via --json) and filesystem non-effect
- [ ] Threat-coverage meta-test fails if a 12.2 row loses its scenario
- [ ] Suite runs in < 120 s on ubuntu CI (fixtures reused via beforeAll where safe)

## Validation

`npm test -- security` green; reviewer spot-checks that assertions would
actually fail on a naive implementation (mutation check on one scenario:
temporarily disable a filter locally → test goes red — documented in PR
description of the implementing PR).

## Dependencies

08, 23, 24, 29 (CLI paths exercised), 12, 13, 20, 21 (mitigations under test).

## Non-goals

Fuzzing (v2); dependency audit tooling (issue 40); prompt-injection content
analysis (v2 per ADR-005).

## Design References

- DESIGN.md 12.2 (threat table — coverage source of truth), 12.3, section 8
