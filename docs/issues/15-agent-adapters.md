# Title

src/adapters: Adapter interface, registry, claude-code and codex adapters, selection logic

## Summary

Implement the adapter layer mapping (runtime, scope) to install base
directories, runtime detection, and the agent-selection precedence used by all
commands.

## Context

DESIGN.md section 13 and ADR-001 fix the interface and the two v1 adapters;
paths are verified in `docs/research/runtime-skill-directories.md`
(`~/.claude/skills`, `.claude/skills`, `~/.codex/skills`, `.codex/skills`).

## Scope

- `src/adapters/types.ts`, `src/adapters/registry.ts`,
  `src/adapters/claude-code.ts`, `src/adapters/codex.ts`
- `tests/adapters/adapters.test.ts`

## Detailed Requirements

1. `types.ts`:
   ```ts
   export interface PlatformEnv { home: string; }   // extend later; keep minimal
   export interface Adapter { id: string; displayName: string;
     detect(p: PlatformEnv): Promise<boolean>;
     baseDir(scope: "user" | "project", roots: { home: string; projectRoot: string | null }): string; // E_CONFIG if project scope with null root
   }
   ```
2. Adapters: `claude-code` → user `join(home, ".claude")`, project
   `join(projectRoot, ".claude")`; `codex` → `.codex` equivalents. `detect` =
   the user base dir exists. Skills dir for materialization =
   `join(baseDir, "skills")` (exposed as `skillsDir(adapter, scope, roots)`
   helper in registry.ts to keep one definition).
3. `registry.ts`:
   ```ts
   export const ADAPTERS: readonly Adapter[];                       // order: claude-code, codex
   export function getAdapter(id: string): Adapter;                 // E_UNKNOWN_AGENT listing known ids
   export function detectAdapters(p: PlatformEnv): Promise<Adapter[]>;
   export function selectAgents(input: { flagAgents: string[] | null; manifestAgents: string[] | null;
     configAgents: string[] | null; detected: Adapter[] }): Adapter[];
   // precedence per DESIGN 13: flag > manifest > config > detected; empty result → E_NO_AGENTS
   ```
4. `selectAgents` validates every id through `getAdapter` (so a typo in any
   layer fails with E_UNKNOWN_AGENT naming the offending source:
   `--agents flag`, `skillet.json`, `config.json`); duplicates removed
   preserving first occurrence; `E_NO_AGENTS` hint explains the four sources
   and suggests `skillet init` / `--agents`.
5. No filesystem writes anywhere in this layer.

## Acceptance Criteria

- [ ] Path table test matches research doc exactly for both adapters × both scopes
- [ ] detect() true/false driven by sandbox dirs (issue 08)
- [ ] Precedence: all 8 combinations of present/absent layers resolve per spec
- [ ] Unknown id in each source position → E_UNKNOWN_AGENT with source named
- [ ] Project scope with null projectRoot → E_CONFIG

## Validation

`npm test -- adapters` green on CI matrix.

## Dependencies

01, 02, 03.

## Non-goals

Materialization (issue 21); additional runtimes (v2, DESIGN 15).

## Design References

- DESIGN.md section 13, ADR-001, docs/research/runtime-skill-directories.md
