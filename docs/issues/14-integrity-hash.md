# Title

core/integrity.ts: canonical tree hash with fixed test vectors

## Summary

Implement the canonical tree hash exactly as specified in DESIGN.md section 8,
over both git tree enumerations and on-disk directories, and freeze test
vectors that index CI, client verify, and future implementations must match.

## Context

The integrity string is the security anchor (ADR-005): index CI computes it at
publish review, the client recomputes after extraction, and `verify` recomputes
from materialized copies. All three must agree bit-for-bit forever; the test
vectors in this issue are therefore normative.

## Scope

- `src/core/integrity.ts`
- `tests/core/integrity.test.ts`
- `tests/fixtures/integrity/` (vector fixtures + expected hashes JSON)

## Detailed Requirements

1. Export:
   ```ts
   export interface HashedFile { relPath: string; cls: "f" | "x"; sha256hex: string; }
   export function integrityFromHashedFiles(files: HashedFile[]): string;   // sorts internally; "sha256-" + base64
   export function hashExtractedEntries(destDir: string, entries: TreeEntry[]): Promise<string>; // cls from git mode
   export function hashLocalDir(dir: string): Promise<string>;              // cls from owner-exec bit; POSIX only meaning — on Windows all "f"
   ```
2. Manifest-string construction exactly per DESIGN 8 step 5:
   `<relPath>\0<cls>\0<sha256hex-lowercase>\n` joined in bytewise-ascending
   UTF-8 order of relPath; final digest = sha256 over the UTF-8 bytes of that
   string; render `"sha256-" + base64(digest)`.
3. `hashLocalDir` enumerates regular files only and throws `E_UNSAFE_TREE` on
   symlinks or non-regular entries (mirrors extraction guarantees for
   `file:` sources and `verify`).
4. Windows note (normative): `cls` is derived from git modes whenever git is
   the source of truth; for local dirs on Windows every file is `"f"`. This
   means `file:` local installs authored with exec bits hash differently across
   OS — document in code comment + DESIGN already marks `file:` as
   non-portable (`portable: false`), so locks never carry cross-OS local hashes.
5. Test vectors (freeze in `tests/fixtures/integrity/vectors.json`):
   - V1: single file `SKILL.md` content `hello\n` class `f` →
     compute once, commit the expected string, and assert literally.
   - V2: three files incl. one `x` class and a nested path.
   - V3: same as V2 with different relPath ordering on input → identical hash.
   - V4: content differing by one byte → different hash (inequality assert).
   - V5: empty file list → hash of empty string (defined, but stage rejects
     empty trees separately).
6. Zero dependencies beyond `node:crypto`, `node:fs`, errors module.

## Acceptance Criteria

- [ ] All five vectors pass; vectors.json contains literal expected integrity strings (no recomputation in assertions)
- [ ] `hashExtractedEntries` over an issue-13 extraction equals `hashLocalDir` over the same dir on POSIX
- [ ] Symlink inside a local dir → E_UNSAFE_TREE
- [ ] Property test (fast-check optional — if used, add as devDependency is NOT allowed; implement a simple 100-case random permutation loop instead): input order never changes the hash

## Validation

`npm test -- integrity` green on CI matrix; vectors byte-identical across OSes.

## Dependencies

01, 02, 13 (TreeEntry type; may land in parallel using the type from 13's branch).

## Non-goals

Limits enforcement (issue 20); verify command (issue 30).

## Design References

- DESIGN.md section 8 (algorithm — normative), ADR-005
