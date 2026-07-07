# Title

core/config.ts: environment, paths, user config, project-root discovery

## Summary

Implement the single module that answers "where is everything": skillet home,
cache dir, index URL, user config file, project root, and env overrides.

## Context

DESIGN.md 7.4 defines the user config and the environment variables
(`SKILLET_HOME`, `SKILLET_INDEX_URL`, `SKILLET_CACHE_DIR`, `SKILLET_YES`).
Testability depends on `SKILLET_HOME` fully sandboxing all writes. Security
invariant (ADR-005 / DESIGN 12.3): the index URL must never be configurable
from project-level files.

## Scope

- `src/core/config.ts`
- `tests/core/config.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface Environment {
     home: string;              // os.homedir() or overridden in tests via HOME
     skilletHome: string;       // SKILLET_HOME ?? join(home, ".skillet")
     cacheDir: string;          // SKILLET_CACHE_DIR ?? userConfig.cacheDir ?? join(skilletHome, "cache")
     indexUrl: string;          // SKILLET_INDEX_URL ?? userConfig.indexUrl ?? DEFAULT_INDEX_URL
     assumeYes: boolean;        // SKILLET_YES === "1"
     userConfigPath: string;    // join(skilletHome, "config.json")
     userManifestPath: string;  // join(skilletHome, "skillet.json")
     userConfig: UserConfig;    // parsed or {} if absent
   }
   export const DEFAULT_INDEX_URL =
     "https://raw.githubusercontent.com/Saber5656/skillet-index/main/index.json";
   export function loadEnvironment(processEnv: NodeJS.ProcessEnv): Environment;
   export function findProjectRoot(cwd: string): string | null; // nearest ancestor containing skillet.json
   export interface ScopeRoots { scope: "project" | "user"; root: string; manifestPath: string; lockfilePath: string; statePath: string; }
   export function scopeRoots(env: Environment, scope: "project" | "user", cwd: string): ScopeRoots; // throws E_CONFIG if project scope and no root found
   ```
2. `scopeRoots` mapping (DESIGN 7.1-7.3): project → `<root>/skillet.json`,
   `<root>/skillet.lock`, `<root>/.skillet/state.json`; user →
   `<skilletHome>/skillet.json`, `<skilletHome>/skillet.lock`,
   `<skilletHome>/state.json`.
3. Malformed user config file → `SkilletError("E_CONFIG", …)` with the file
   path and a hint to fix or delete it. Unknown keys are ignored with no error
   (forward compatibility); zod schema arrives in issue 05 — until then parse
   with a local minimal validator and refactor to the shared schema in issue 05
   (leave a `TODO(issue-05)` marker).
4. `findProjectRoot` stops at filesystem root; does NOT fall back to git root.
5. Tilde expansion for `cacheDir` from config (`~/x` → `join(home, "x")`).
6. Pure with respect to inputs: no reads of `process.env` at import time; all
   filesystem access is lazy and synchronous (`node:fs`).

## Acceptance Criteria

- [ ] With `SKILLET_HOME=/tmp/sandbox`, every derived path is under `/tmp/sandbox` (except cacheDir/indexUrl overrides)
- [ ] Env vars beat user config; user config beats defaults (three-layer test per field)
- [ ] `findProjectRoot` finds skillet.json two levels up; returns null when absent
- [ ] Project `scopeRoots` without a project root throws `E_CONFIG` with hint mentioning `skillet init`
- [ ] Nothing in this module reads project files to determine `indexUrl`

## Validation

`npm test -- config` green on macOS/Linux/Windows CI (path handling via
`node:path`, no hardcoded `/`).

## Dependencies

01-project-scaffold, 02-error-taxonomy.

## Non-goals

Writing config files; CLI flags parsing (`--scope`, `-C`) — issue 17.

## Design References

- DESIGN.md 7.4 (config/env), 7.1-7.3 (file locations), 12.3 (index URL invariant)
