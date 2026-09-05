# Title

cli/index.ts: commander program, global flags, error boundary, command registry

## Summary

Implement the real CLI entry: global flags, reporter construction, environment
loading, the top-level error boundary mapping SkilletError → exit codes, and
the registration point where each command issue plugs in.

## Context

DESIGN.md section 14 fixes global flags, the `--json` envelope discipline, and
the exit-code contract (issue 02). Command implementations arrive in later
issues; this issue ships the skeleton plus `--version`/`--help`.

## Scope

- `src/cli/index.ts`, `src/cli/context.ts`
- `tests/cli/entry.test.ts` (spawns built CLI via issue-08 sandbox)

## Detailed Requirements

1. Global flags (commander, `enablePositionalOptions`):
   `--json`, `--yes/-y`, `--verbose`, `--no-color`, `-C <dir>`,
   `--scope <project|user>` (default `project`), `-g` (alias for
   `--scope user`), `--agents <ids...>` (comma or repeated). Invalid enum
   values → E_USAGE (commander override, not commander's default exit 1).
2. `src/cli/context.ts`:
   ```ts
   export interface CmdContext { env: Environment; reporter: Reporter;
     cwd: string;                    // after -C applied
     scope: "project" | "user"; flagAgents: string[] | null;
     assumeYes: boolean;             // --yes || SKILLET_YES
     json: boolean; }
   export function buildContext(opts: GlobalOpts): CmdContext;
   ```
   Color rule: `!--no-color && !NO_COLOR && stderr.isTTY` (human text goes to
   stderr in json mode, stdout otherwise — reporter handles it, issue 04).
3. Error boundary wraps every command action:
   `catch (e) → reporter.jsonError(cmd, e) | reporter.error(msg, hint)` then
   `process.exitCode = exitCodeFor(code)`; non-SkilletError additionally prints
   stack when `--verbose`. Never `process.exit()` inside commands (streams must
   flush).
4. Command registry: `registerCommands(program, buildContext)` imports from
   `src/cli/commands/*.ts`; each command module exports
   `register(program: Command, ctx: () => CmdContext): void`. This issue adds
   placeholder registrations that throw `E_INTERNAL "not implemented"` for:
   init, add, install, update, remove, list, search, info, verify, validate,
   cache, index — so `--help` already shows the full v1 surface with one-line
   descriptions copied from DESIGN 14's table.
5. `--version` prints the version only (stdout, exit 0). Unknown command →
   E_USAGE exit 2 with help hint.
6. `-C <dir>`: `process.chdir` before context build; missing dir → E_USAGE.

## Acceptance Criteria

- [ ] `skillet --help` lists all 12 commands with descriptions; exit 0
- [ ] `skillet nope` → exit 2; stderr contains "unknown command"
- [ ] `skillet install --json` (placeholder) → stdout is the exact error envelope, exit 1, and nothing else on stdout
- [ ] `--scope bananas` → exit 2
- [ ] `-C <tmpdir>` changes resolution of project root (asserted via a fixture manifest)
- [ ] NO_COLOR / --no-color produce ANSI-free stderr

## Validation

`npm test -- entry` green; `node dist/cli.js --help` manually sane.

## Dependencies

01, 02, 03, 04, 08.

## Non-goals

Any real command behavior (issues 18, 23-32, 35).

## Design References

- DESIGN.md section 14 (global flags, envelope, exit codes)
