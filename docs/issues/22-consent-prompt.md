# Title

core/consent.ts: install consent prompt with script/tool transparency

## Summary

Implement the consent gate shown before first materialization of a package:
render the security-relevant summary (source, files, executables,
allowed-tools, license, deprecation) and collect an explicit yes/no, honoring
`--yes`/`SKILLET_YES` and failing closed on non-TTY.

## Context

DESIGN.md 12.4 fixes the prompt content and the fail-closed rule (exit 5 /
`E_CONSENT_DECLINED`). DESIGN 10.1: consent is skipped for reinstalls of an
identical (package, version, integrity) triple already recorded in state
(same-content re-materialization is not a new trust decision). Deprecated
packages require confirmation even with a warm cache (DESIGN 9).

## Scope

- `src/core/consent.ts`
- `tests/core/consent.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface ConsentSummary { id: string; version: string;
     sourceLabel: string;            // e.g. "index: github.com/anthropics/skills @ 0123456" | "git: gitlab.com/... @ abc1234" | "path: ../dev-skill"
     fileCount: number; totalBytes: number;
     executables: string[]; allowedTools?: string; license?: string;
     targets: string[];              // e.g. ["claude-code (project)", "codex (project)"]
     deprecated?: { reason: string; replacement: string | null } }
   export function needsConsent(summary: ConsentSummary, state: State): boolean;
   export function requestConsent(summary: ConsentSummary, io: { reporter: Reporter;
     assumeYes: boolean; isTTY: boolean; readLine: () => Promise<string> }): Promise<void>; // resolves or throws E_CONSENT_DECLINED
   ```
2. Rendering (via reporter, stderr): exactly the DESIGN 12.4 shape — header
   line `id@version  (sourceLabel)`, indented facts (files+size human-readable
   KiB/MiB, executables listed up to 5 then `… and N more`, allowed-tools
   verbatim, license), deprecation in yellow with reason/replacement, then
   `Install to: <targets>? [y/N]`.
3. Decision rules:
   - `assumeYes` → accepted without prompting, but the summary is still
     printed (auditability in CI logs) with `(auto-approved: --yes)`.
   - Not a TTY and not assumeYes → `E_CONSENT_DECLINED` with hint about
     `--yes` / `SKILLET_YES=1`.
   - TTY: accept only `y`/`yes` (case-insensitive, trimmed); everything else
     (incl. empty) declines.
4. `needsConsent`: false only when state holds a receipt with the same
   (package, version, integrity) for ANY agent AND `deprecated` is undefined;
   deprecated always re-prompts.
5. `readLine` injected for testability; the CLI passes a readline-on-stdin
   implementation (built here as `stdinReadLine()` export, used by issue 23).
6. No timeouts on the prompt (human decision); Ctrl-C propagates as normal
   SIGINT (no special handling).

## Acceptance Criteria

- [ ] Snapshot test of the rendered block for: minimal package; package with 7 executables + allowed-tools; deprecated package
- [ ] Decision matrix: (assumeYes × tty × answer) — 6 cases with expected resolve/throw
- [ ] needsConsent false for identical triple; true when integrity differs, version differs, or deprecated
- [ ] Auto-approve still prints the summary
- [ ] Declined → E_CONSENT_DECLINED (exit code 5 via mapping)

## Validation

`npm test -- consent` green.

## Dependencies

02, 04, 16.

## Non-goals

Persisting consent decisions beyond state receipts (no TOFU database in v1);
per-file diff display (v2).

## Design References

- DESIGN.md 12.4 (prompt — normative), 10.1 (CONSENT phase), 9 (deprecation)
