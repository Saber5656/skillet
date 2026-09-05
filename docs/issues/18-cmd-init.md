# Title

skillet init: create manifest, detect agents, gitignore state dir

## Summary

Implement `skillet init [--agents ids…] [-g]`: create the scope's manifest
with detected or specified agents, and (project scope) append `.skillet/` to
`.gitignore`.

## Context

DESIGN.md 14 (command table) and 7.1 (manifest). Init is the entry point of
the team workflow (workflow 1 in DESIGN 4). Project-root subtlety: before a
manifest exists, `findProjectRoot` returns null — init defines the root as the
current working directory (documented behavior).

## Scope

- `src/cli/commands/init.ts`
- `tests/cli/init.test.ts`

## Detailed Requirements

1. Behavior (project scope, default): target = `<cwd>/skillet.json`.
   - Exists already → E_USAGE "already initialized" with path (exit 2), no changes.
   - Agents: `--agents` if given (validated via `getAdapter`), else detected
     set (issue 15), else E_NO_AGENTS with hint listing supported ids and
     `--agents` usage.
   - Write manifest via `newManifest(agents)` (issue 09).
   - `.gitignore` handling: create or append a line `.skillet/` if not already
     present (exact-line match; preserve file's existing newline style; append
     with a preceding comment line `# skillet local state` when appending to a
     non-empty file).
   - If inside a nested dir of an existing project (an ancestor has
     skillet.json), warn (`warning: <ancestor> already contains skillet.json`)
     but proceed (monorepo sub-project is legitimate).
2. User scope (`-g`): target `~/.skillet/skillet.json` (env.userManifestPath);
   no gitignore handling; create `skilletHome` dir as needed.
3. Output: human — summary lines (created path, agents, gitignore action);
   json — `{ manifestPath, agents, gitignoreUpdated: boolean }`.
4. Idempotence guarantees: no partial writes (manifest write is atomic;
   gitignore append happens only after manifest success).

## Acceptance Criteria

- [ ] Fresh dir: creates manifest with detected agents (sandbox provides `~/.claude` only → `["claude-code"]`)
- [ ] `--agents codex` overrides detection; unknown id → exit 2 naming the flag
- [ ] Second run → exit 2, files untouched (mtime compare)
- [ ] `.gitignore` created when absent; appended without duplication when run in a repo that already ignores `.skillet/`
- [ ] `-g` writes under SKILLET_HOME sandbox and touches no project files
- [ ] `--json` output matches the schema above

## Validation

`npm test -- init` green; manual smoke in a temp dir.

## Dependencies

02, 03, 04, 09, 15, 17.

## Non-goals

Installing anything; interactive agent selection prompts (flags/detection only
in v1).

## Design References

- DESIGN.md 4 (workflow 1), 7.1, 13 (selection), 14 (command table)
