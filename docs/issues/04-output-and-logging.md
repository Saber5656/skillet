# Title

core/log.ts: reporter with human output, tables, and the --json envelope

## Summary

Implement the output layer every command uses: human-readable messages and
tables to stderr/stdout with color/TTY handling, and a machine `--json` mode
with the fixed result envelope.

## Context

DESIGN.md section 14: in `--json` mode, stdout carries exactly one JSON
document `{ ok, command, data, error }`; all human text goes to stderr. Human
mode prints results to stdout, diagnostics to stderr. Colors respect
`--no-color` and the `NO_COLOR` env var.

## Scope

- `src/core/log.ts`
- `tests/core/log.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface ReporterOptions { json: boolean; color: boolean; verbose: boolean;
     stdout?: NodeJS.WritableStream; stderr?: NodeJS.WritableStream; }
   export interface Reporter {
     info(msg: string): void;            // stderr in json mode? NO -> suppressed in json mode unless verbose
     warn(msg: string): void;            // stderr, prefixed "warning:", yellow
     error(msg: string, hint?: string): void; // stderr, prefixed "error:", red; hint dimmed on next line
     verbose(msg: string): void;         // stderr only when verbose
     out(msg: string): void;             // stdout, human results (no-op in json mode)
     table(rows: string[][], header?: string[]): void; // aligned columns, no borders; no-op in json mode
     jsonResult(command: string, data: unknown): void;    // stdout: { ok: true, command, data, error: null }
     jsonError(command: string, err: unknown): void;      // stdout: { ok: false, command, data: null, error: toErrorPayload(err) }
   }
   export function createReporter(opts: ReporterOptions): Reporter;
   ```
2. In json mode: `info`/`warn` go to stderr as plain text (warnings must remain
   visible to CI logs), `out`/`table` are no-ops, and exactly one of
   `jsonResult`/`jsonError` may be called per process (second call throws
   `E_INTERNAL` — guards against double output).
3. Color via `picocolors`, enabled only when `opts.color` is true; the caller
   (issue 17) computes `color = !flagNoColor && !env.NO_COLOR && isTTY`.
4. `table` column widths from max cell width, two-space gutter, header
   underlined with dashes; deterministic output (tested with fixed input).
5. JSON output is `JSON.stringify(value, null, 2)` + trailing newline.
6. No direct `console.*` anywhere; streams only (default `process.stdout/err`).

## Acceptance Criteria

- [ ] json mode with a result: stdout parses as the exact envelope; stderr may contain warnings
- [ ] human mode: `out` writes stdout; `error("x","do y")` writes two stderr lines
- [ ] `NO_COLOR`-driven `color:false` output contains zero ANSI escapes (regex test)
- [ ] Double `jsonResult` call throws
- [ ] `table` snapshot test with 3×3 rows incl. wide CJK-free ASCII content

## Validation

`npm test -- log` green; manual check `node dist/cli.js --version` unaffected.

## Dependencies

01-project-scaffold, 02-error-taxonomy.

## Non-goals

Flag parsing and reporter construction wiring (issue 17); progress spinners
(not in v1).

## Design References

- DESIGN.md section 14 (--json envelope, output conventions)
