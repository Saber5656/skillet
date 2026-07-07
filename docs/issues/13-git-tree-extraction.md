# Title

core/gitfetch.ts (part 2): tree enumeration and filtered blob extraction

## Summary

Implement `listTree` (enumerate a skill subtree at a commit via `git ls-tree`)
and `extractTree` (write blobs to a destination via `git cat-file --batch`),
the structurally-safe alternative to archive unpacking.

## Context

ADR-008 / DESIGN.md 10.1 EXTRACT phase and section 8 step 1: entries are
enumerated and type/mode-checked BEFORE any byte is written; symlinks,
gitlinks, and traversal names never reach the filesystem. The same enumeration
feeds integrity hashing (issue 14) so extraction and hashing cannot diverge.

## Scope

- `src/core/gitfetch.ts` (add tree functions)
- `tests/core/gitfetch-tree.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface TreeEntry { relPath: string;    // relative to the skill dir, forward slashes
     mode: "100644" | "100755" | "120000" | "160000" | string;
     type: "blob" | "commit" | "tree"; oid: string; size: number; }
   export function listTree(env: Environment, url: string, commit: string, path: string): Promise<TreeEntry[]>;
   export function extractTree(env: Environment, url: string, commit: string, path: string,
     entries: TreeEntry[], destDir: string): Promise<void>;
   ```
2. `listTree` runs
   `git ls-tree -r -l -z <commit> -- <path>` in the mirror (empty `path` → tree
   root), parses NUL-delimited records `<mode> <type> <oid> <size>\t<fullpath>`,
   strips the `path` prefix to produce `relPath`. `path` must already satisfy
   relPath rules (assert; defense in depth). Missing/empty subtree →
   `E_RESOLVE` ("no skill directory at <path> in <shortsha>").
3. `listTree` performs NO filtering (returns raw truth); the safety filter
   lives in stage (issue 20) so index CI can report ALL violations. But
   `extractTree` MUST refuse (E_UNSAFE_TREE) if any entry is not
   `type === "blob"` with mode `100644`/`100755`, or any relPath fails relPath
   rules or case-fold-duplicates another — extraction is unconditionally safe
   even if a caller forgets to filter.
4. `extractTree` streams `git cat-file --batch` on the mirror: writes each blob
   to `destDir/<relPath>` (mkdir -p parents; parents must stay within destDir —
   verify with `path.resolve` prefix check), sets mode 0o755 for `100755` on
   POSIX (no-op on Windows), verifies byte length against `size`, and verifies
   the oid ordering matches the request sequence. Any mismatch → E_INTEGRITY.
5. Concatenated batch protocol handling: request all oids on stdin
   (`<oid>\n`…), parse `<oid> <type> <size>\n<raw bytes>\n` frames; a `missing`
   response → E_FETCH.
6. Determinism: after extraction, re-`lstat` every written file — any symlink
   found (race/tamper) → E_UNSAFE_TREE and destDir is deleted before throwing.
   destDir must be empty before starting (E_INTERNAL otherwise).

## Acceptance Criteria

- [ ] Fixture repo with nested dirs + one executable script: extraction reproduces content, layout, and the exec bit (POSIX)
- [ ] Fixture with a symlink entry: `listTree` reports it; `extractTree` refuses with E_UNSAFE_TREE and writes nothing
- [ ] Fixture with a submodule (gitlink): same refusal
- [ ] Path prefix stripping: skill at `skills/pdf` yields relPaths without the prefix
- [ ] Case-fold duplicate (`README.md` + `readme.md`) → E_UNSAFE_TREE (created via git plumbing on a case-sensitive runner; test skipped on case-insensitive FS with an explicit skip reason)
- [ ] Byte-size mismatch injected via mock runner → E_INTEGRITY, destDir cleaned

## Validation

`npm test -- gitfetch-tree` green on CI matrix.

## Dependencies

01, 02, 03, 08, 12.

## Non-goals

Limits (file count/total size — issue 20); hashing (issue 14); local-path
staging (issue 20).

## Design References

- DESIGN.md section 8 step 1 (rejection list), 10.1 EXTRACT, 12.2 (zip-slip row), ADR-008
