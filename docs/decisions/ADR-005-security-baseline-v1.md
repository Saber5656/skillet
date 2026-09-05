# ADR-005: v1 security baseline — pin, hash, never execute; signing deferred

- Status: Accepted (conservative default, 2026-07-07)
- Deciders: Fable (per task instruction to take conservative security defaults)

## Context

Skills are prompt+script payloads installed into agent runtimes; they are a
supply-chain vector (typosquatting, moved tags, compromised sources, path
traversal payloads, malicious scripts). Full cryptographic signing (sigstore)
would materially expand v1 (key management, verification infra) without which a
strong-but-simpler baseline is still achievable.

## Decision

v1 baseline (normative, DESIGN.md section 12):

1. Every resolved package pins a 40-hex commit AND a sha256 canonical tree hash;
   client refuses content whose recomputed hash differs (index-declared hash is
   authoritative for registry installs; lockfile hash for reinstalls).
2. Published versions are immutable in the index (CI rejects edits).
3. skillet never executes fetched content: no install hooks of any kind, by design.
4. Extraction is enumerated (`git ls-tree`), not archive-unpacked: symlinks,
   gitlinks, traversal names, case-fold duplicates, and oversize trees are
   rejected before any write.
5. Explicit consent before first materialization (script/allowed-tools
   transparency); non-TTY fails closed without `--yes`/`SKILLET_YES`.
6. Deletion/overwrite only of receipt-tracked (state.json) directories.
7. Index URL is user-level config only — never project-level (a hostile repo
   cannot repoint the registry).
8. Signing of index snapshots/packages, and content scanning, are deferred to v2;
   SECURITY.md states this honestly.

## Consequences

- Positive: strong practical guarantees with zero key infrastructure; every
  control is mechanically testable (security test suite, issue 37).
- Negative: a compromised GitHub account controlling the index repo remains a
  root-of-trust risk until signing lands (documented residual risk).
