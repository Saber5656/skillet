# ADR-003: TypeScript / Node.js, distributed on npm

- Status: Accepted (user decision, 2026-07-07)
- Deciders: Saber5656 (option choice), Fable (design)

## Context

Candidates: TypeScript/Node (npm), Go, Rust (single binaries). The entire target
audience (Claude Code / Codex users) already has Node installed; issues will be
implemented mostly by lower-capability coding agents, whose success rate and
test ergonomics are best in TypeScript.

## Decision

TypeScript, Node >= 20, ESM-only, bundled with tsup, tested with vitest,
published to npm as `@skillet/cli` with bin `skillet` (name per ADR-006).
Runtime dependency budget is fixed at: `commander`, `zod`, `yaml`, `semver`,
`picocolors`; anything beyond requires a new ADR (supply-chain surface control).
Child processes via `node:child_process` wrapper (no execa); HTTP via global fetch.

## Consequences

- Positive: `npm i -g` / `npx` distribution; highest mechanical-implementability
  for weak agents; zod schemas double as validation + JSON Schema exports.
- Negative: not a single static binary; Node startup latency (~50-100 ms) —
  acceptable for a package manager; Homebrew formula deferred.
