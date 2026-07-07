# ADR-001: Multi-agent core with runtime adapters

- Status: Accepted (user decision, 2026-07-07)
- Deciders: Saber5656 (option choice), Fable (design)

## Context

Skills follow the Agent Skills open standard and are consumed by multiple
runtimes (Claude Code, Codex, and a growing list). A Claude-Code-only manager
would compete directly with the first-party plugin marketplace and have weak
standalone value. The user selected multi-agent support for v1.

## Decision

The core (resolution, fetching, integrity, lockfile, state) is runtime-agnostic
and operates on spec-conformant skill directories. Runtime specifics are isolated
behind a minimal `Adapter` interface (id, detection, base directory per scope).
v1 ships exactly two adapters: `claude-code` and `codex`
(paths verified in `docs/research/runtime-skill-directories.md`).

## Consequences

- Positive: differentiated vs first-party marketplaces; future adapters (Cursor,
  OpenCode, …) are additive files, not refactors; core testable without any runtime.
- Negative: v1 carries two adapters' test surface; agent targeting UX
  (`--agents`, manifest `agents`) must exist from day one.
- Lockfile stays adapter-free (pins content, not destinations), so adding
  adapters never invalidates locks.
