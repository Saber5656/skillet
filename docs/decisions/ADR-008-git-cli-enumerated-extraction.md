# ADR-008: Fetch via git CLI; extract via enumerated ls-tree/cat-file (no archives)

- Status: Accepted (conservative default, 2026-07-07)
- Deciders: Fable

## Context

Payloads live in git repos. Options: GitHub tarball API (no git dependency, but
GitHub-only and needs a tar parser — an attack surface of its own), libgit2
bindings (native module pain), or the user's `git` CLI (host-agnostic, respects
the user's existing credential helpers for private sources).

## Decision

Require `git >= 2.30` on PATH (checked with a clear error). Maintain a bare
mirror per source URL under the cache; ensure commits via `git fetch origin
<sha>` with a documented fallback chain (DESIGN 10.2). Extraction never unpacks
an archive: `git ls-tree -r` enumerates entries first, safety filters reject
symlinks/gitlinks/traversal/oversize (DESIGN section 8), then blobs are written
via `git cat-file --batch`. All spawns are argument-vector (no shell), with
scrubbed env, protocol allowlist, timeouts, and `--` pathspec separators.

## Consequences

- Positive: works with GitHub/GitLab/Codeberg/self-hosted over https or ssh;
  private sources work through ambient credentials with zero skillet credential
  code; the zip-slip vulnerability class is structurally absent; extraction and
  integrity share one enumeration path.
- Negative: hard runtime dependency on git presence/version; per-blob extraction
  is slower than tar for huge trees (bounded by the 1000-file/20 MiB policy);
  SHA-fetch capability varies by host (fallback chain covers it).
