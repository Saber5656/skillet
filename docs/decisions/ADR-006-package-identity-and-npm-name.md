# ADR-006: Owner-bound package identity; npm distribution name `@skillet/cli`

- Status: Accepted (user decision on npm name, 2026-07-07)
- Deciders: Saber5656 (npm name), Fable (identity scheme)

## Context

Flat names (Homebrew-style) are squat-prone. npm research
(`docs/research/prior-art-and-naming.md`) found `skillet` and `skillet-cli`
taken (the latter active and domain-adjacent), while the `@skillet` scope has
zero published packages. The user rejected suffix names (`skillet-pm`) and chose
the scoped name.

## Decision

1. **Package identity** in the skillet registry is `owner/skill-name` where
   `owner` MUST equal the source repository's owner (validated by index CI,
   case-insensitively; stored lowercase). `skill-name` follows Agent Skills spec
   name rules and must equal SKILL.md `name` and the directory basename.
2. **npm distribution name** is `@skillet/cli`, bin `skillet`. Registering the
   npm org "skillet" is a manual user action; if unavailable, fallback name is
   `skillets` (verified free 2026-07-07) — a one-string change confined to
   issue 01 scaffolding and issue 39/40 docs/release.

## Consequences

- Positive: names carry provenance (owner == repo owner) — typosquatting a
  popular owner requires controlling that owner's GitHub account; identity is
  mechanically CI-checkable.
- Negative: skill-name basename collisions across owners cannot coexist in one
  scope (runtime constraint: dir == name); resolver must reject with
  `E_NAME_CONFLICT`. Brand adjacency with dafrick/skillet-cli remains and is
  handled by positioning, not naming.
