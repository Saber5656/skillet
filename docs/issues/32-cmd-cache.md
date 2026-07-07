# Title

skillet cache: dir and clean subcommands

## Summary

Implement `skillet cache dir` (print the cache location) and `skillet cache
clean` (delete mirrors, index cache, and orphaned stage dirs — never
installed skills).

## Context

DESIGN 14 command table; cache layout from DESIGN 10.2 / 7.6 / issue 20
(`<cacheDir>/repos/*`, `<cacheDir>/index/*`, `<cacheDir>/stage/*`). Clean must
be safe under a concurrent skillet process (mirror locks, issue 12).

## Scope

- `src/cli/commands/cache.ts`
- `tests/cli/cache.test.ts`

## Detailed Requirements

1. `skillet cache dir`: prints the absolute cache dir (stdout, single line,
   no decoration — script-friendly); `--json` → `{ dir }`. Exit 0 even if the
   dir does not exist yet.
2. `skillet cache clean [--stage-only]`:
   - Default: remove `repos/`, `index/`, `stage/` subtrees (recreate empty
     `cacheDir`); print freed bytes (computed before deletion).
   - `--stage-only`: remove only `stage/` entries older than 1 hour (mtime) —
     the orphan reaper referenced by issue 20.
   - Respect mirror locks: a locked mirror (issue 12 lock protocol) is skipped
     with a warning naming the lock path and the crashed-process hint; exit 0
     with skips, since clean is best-effort.
   - MUST refuse (E_CONFIG) if `cacheDir` resolves outside `skilletHome` AND
     is not the configured override — i.e., sanity-check the path is one of
     the two legitimate derivations before `rm -rf` (defense against config
     corruption pointing at `/` — assert the resolved dir's basename chain or
     that it equals the configured value verbatim).
3. Never touches manifests, lockfiles, state, or agent dirs (test-asserted).
4. `--json` for clean: `{ freedBytes, removed: {repos, index, stage},
   skipped: [{path, reason}] }`.

## Acceptance Criteria

- [ ] `cache dir` prints the sandbox path; works before first cache write
- [ ] Warm cache → clean removes all three subtrees; subsequent `search` re-fetches (proves index cache gone)
- [ ] `--stage-only` removes old stage dirs, keeps fresh ones
- [ ] Locked mirror skipped with warning; other content removed
- [ ] Corrupted config pointing cacheDir at a temp dir outside legitimate derivations → E_CONFIG, nothing deleted
- [ ] Installed skills and state untouched (fixture assertion)

## Validation

`npm test -- cache` green.

## Dependencies

03, 04, 12 (lock protocol), 17, 20 (stage layout).

## Non-goals

Size-based eviction policies (v2); per-repo selective eviction (v2).

## Design References

- DESIGN.md 14 (command), 10.2 (mirror layout), 7.6 (index cache)
