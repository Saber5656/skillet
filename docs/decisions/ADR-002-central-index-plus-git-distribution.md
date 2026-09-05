# ADR-002: Central index repository + git-sourced payloads

- Status: Accepted (user decision, 2026-07-07)
- Deciders: Saber5656 (option choice), Fable (design)

## Context

README requires search and version management. Alternatives considered:
(a) git-direct only (zero infra, but search is weak and versions are just refs),
(b) hosted registry server (best capabilities, but server cost/auth/ops are
disproportionate for v1), (c) curated index repo on GitHub + payloads fetched
from source git repos (Homebrew-core-like).

## Decision

Option (c). A separate GitHub repository (`skillet-index`) holds one JSON
metadata file per package (`registry/<owner>/<name>.json`) mapping semver
versions to `{commit, path, integrity}` in the source repo. CI builds a single
aggregate `index.json` that clients fetch over HTTPS with ETag caching.
Direct git installs remain supported and bypass the index (pseudo-versions).

## Consequences

- Positive: real search + real versions with zero servers; publishing is a PR;
  the index is auditable git history; offline-capable after one fetch.
- Negative: curation labor; aggregate file has a scaling ceiling (known unknown,
  sharding deferred to v2); index availability depends on GitHub raw hosting.
- Version immutability becomes a CI-enforced security invariant (published
  `commit`/`integrity` can never change for a released version).
