# ADR-004: Publishing = manual PR to the index repo, validated by CI

- Status: Accepted (user decision, 2026-07-07)
- Deciders: Saber5656 (option choice), Fable (design)

## Context

Skill authors need a path into the registry. A `skillet publish` command (fork +
PR automation) adds GitHub authentication, token handling, and a write-capable
security boundary to v1. A consume-only v1 (maintainer edits index by hand) would
strangle ecosystem growth.

## Decision

v1 publishing flow: the author edits/adds `registry/<owner>/<name>.json` in the
index repo and opens a PR by hand (CONTRIBUTING.md documents the exact steps,
including how to compute `integrity` via `skillet validate --print-integrity`).
Index CI (`skillet index validate`) gives a deterministic verdict; maintainers
merge. The skillet CLI holds no GitHub credentials and has no write path.

## Consequences

- Positive: v1 has zero credential handling; every registry change is a reviewed,
  attributable PR; CI logic and local validation are literally the same code.
- Negative: author friction (manual JSON + PR); acceptable for early ecosystem
  size. `skillet publish` automation is the top v2 candidate and slots in without
  changing the index format.
