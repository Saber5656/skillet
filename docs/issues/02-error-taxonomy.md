# Title

core/errors.ts: SkilletError taxonomy and exit-code contract

## Summary

Implement the single error type, the closed set of error codes, and the fixed
exit-code mapping that every module and command uses.

## Context

DESIGN.md section 14 fixes exit codes as a public contract (scripts and CI
depend on them). All user-facing failures must carry a stable `E_*` code, a
human message, and an optional remediation hint; `--json` mode serializes them.

## Scope

- `src/core/errors.ts`
- `tests/core/errors.test.ts`

## Detailed Requirements

1. Export `type ErrorCode` as a string-literal union, exactly:
   `E_USAGE`, `E_CONFIG`, `E_RESOLVE`, `E_UNKNOWN_PACKAGE`,
   `E_NO_MATCHING_VERSION`, `E_NAME_CONFLICT`, `E_FETCH`, `E_GIT_MISSING`,
   `E_UNSAFE_TREE`, `E_INTEGRITY`, `E_SPEC_INVALID`, `E_CONSENT_DECLINED`,
   `E_TARGET_CONFLICT`, `E_MATERIALIZE`, `E_FROZEN_DRIFT`, `E_NO_AGENTS`,
   `E_UNKNOWN_AGENT`, `E_DRIFT`, `E_CACHE`, `E_INTERNAL`.
2. Export `class SkilletError extends Error` with fields
   `{ code: ErrorCode; hint?: string; cause?: unknown }` and constructor
   `(code, message, opts?: { hint?; cause? })`.
3. Export `exitCodeFor(code: ErrorCode): number` implementing DESIGN 14:
   - 2: `E_USAGE`, `E_CONFIG`, `E_UNKNOWN_AGENT`
   - 3: `E_RESOLVE`, `E_UNKNOWN_PACKAGE`, `E_NO_MATCHING_VERSION`,
        `E_NAME_CONFLICT`, `E_FETCH`, `E_GIT_MISSING`, `E_FROZEN_DRIFT`,
        `E_NO_AGENTS`, `E_CACHE`
   - 4: `E_UNSAFE_TREE`, `E_INTEGRITY`, `E_SPEC_INVALID`, `E_TARGET_CONFLICT`,
        `E_MATERIALIZE`
   - 5: `E_CONSENT_DECLINED`
   - 6: `E_DRIFT`
   - 1: `E_INTERNAL` (and the default for non-SkilletError throws)
4. Export `toErrorPayload(err: unknown): { code; message; hint? }` used by the
   `--json` envelope (unknown errors → `E_INTERNAL`, message without stack).
5. No dependencies on other core modules (this module is a leaf).

## Acceptance Criteria

- [ ] Every code above maps to exactly the specified exit code (table-driven test)
- [ ] `toErrorPayload(new RangeError("x"))` yields code `E_INTERNAL`
- [ ] `SkilletError` instances survive `instanceof` after ESM import from two paths
- [ ] Module imports nothing from `src/` (leaf check via eslint import rule or test)

## Validation

`npm test -- errors` green; typecheck passes.

## Dependencies

01-project-scaffold.

## Non-goals

Rendering/formatting of errors (issue 04); process.exit handling (issue 17).

## Design References

- DESIGN.md section 14 (exit codes), section 10.1 (failure phases)
