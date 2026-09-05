# Title

core/state.ts: per-scope install receipts (state.json)

## Summary

Implement reading/writing the local state file that records exactly what
skillet materialized where — the safety foundation for replace, remove, list,
and verify.

## Context

DESIGN.md 7.3 defines the format; ADR-007 makes receipts the precondition for
any destructive operation ("skillet only deletes what state records").
State is local-only (never committed); `skillet init` gitignores `.skillet/`.

## Scope

- `src/core/state.ts`
- `tests/core/state.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export function readState(statePath: string): State;               // missing file → { stateVersion: 1, installs: [] }; invalid → E_CONFIG with hint (delete to reset; skillet re-verifies via hashes)
   export function writeState(statePath: string, s: State): void;     // atomic + stable (jsonio), parent dir created
   export interface InstallReceipt { package: string; version: string; integrity: string;
     agent: string; dir: string; installedAt: string; }
   export function upsertReceipt(s: State, r: InstallReceipt): State;  // keyed by (package, agent); pure
   export function removeReceipt(s: State, pkg: string, agent: string): State;
   export function findReceipts(s: State, q: { pkg?: string; agent?: string }): InstallReceipt[];
   export function receiptForDir(s: State, dir: string): InstallReceipt | null; // normalized comparison
   ```
2. `installs` array kept sorted by (package, agent) for stable serialization;
   `dir` stored with forward slashes, relative to the scope root (DESIGN 7.3) —
   normalization helpers included and used by materializer (issue 21).
3. `installedAt` is an ISO-8601 UTC string provided by the caller (state.ts
   itself never calls `Date.now()` — testability).
4. Corrupt state handling is explicitly lenient-but-loud: `E_CONFIG` error
   text explains that deleting the file only loses receipts (skillet will then
   refuse to touch previously-managed dirs until re-installed) — this exact
   consequence must be in the hint.
5. stateVersion mismatch → E_CONFIG with upgrade hint (same pattern as lockfile).

## Acceptance Criteria

- [ ] Round-trip + byte-stability tests (two writes identical)
- [ ] upsert replaces the (package, agent) pair without duplicating; ordering maintained
- [ ] receiptForDir matches regardless of `\` vs `/` input on Windows
- [ ] Missing file yields empty state; corrupt file → E_CONFIG with documented hint text
- [ ] No `Date.now()` usage (grep test or injected-clock design)

## Validation

`npm test -- state` green on CI matrix.

## Dependencies

01, 02, 03, 05, 09 (jsonio).

## Non-goals

Deciding when receipts are written/removed (issues 21/23/29); drift detection
logic (issue 30).

## Design References

- DESIGN.md 7.3 (format), 10.3 (managed-only mutations), ADR-007
