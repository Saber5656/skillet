# Title

core/names.ts: parse/format package ids, dependency specifiers, and add-refs

## Summary

Implement the grammar layer converting user-typed strings and manifest values
into structured references (and back), used by add/install/update/remove and
the resolver.

## Context

DESIGN.md 6.1-6.2 defines: package id `owner/skill-name`; manifest specifiers
(semver range, `github:` shorthand, object git form, `file:`); and add-time
refs (`owner/name[@range]`, `github:owner/repo[/path…]#ref`, git URL object
equivalent, `file:` path). Direct-git package ids derive the owner from the
URL and the skill name from SKILL.md after staging.

## Scope

- `src/core/names.ts`
- `tests/core/names.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export interface PackageId { owner: string; skill: string; }        // both validated
   export function parsePackageId(s: string): PackageId;               // E_USAGE on failure
   export function formatPackageId(id: PackageId): string;             // "owner/skill"

   export type ParsedSpecifier =
     | { kind: "range"; range: string }                                 // validRange or "latest"
     | { kind: "github"; owner: string; repo: string; path: string; ref: string }
     | { kind: "git"; url: string; path: string; ref: string }
     | { kind: "path"; path: string };
   export function parseSpecifier(value: unknown): ParsedSpecifier;     // manifest value; E_CONFIG on failure

   export type AddRef =
     | { kind: "registry"; id: PackageId; range: string }               // default range "latest"
     | { kind: "github"; owner: string; repo: string; path: string; ref: string | null } // null → resolve HEAD at add time
     | { kind: "git"; url: string; path: string; ref: string | null }
     | { kind: "path"; path: string };
   export function parseAddRef(s: string): AddRef;                      // E_USAGE on failure
   export function specifierForLock(ref: AddRef, resolvedSha: string): ParsedSpecifier; // SHA-pinned manifest value per DESIGN 6.2
   ```
2. `github:` shorthand grammar:
   `github:<owner>/<repo>(/<path…>)?(#<ref>)?` — owner/repo follow GitHub rules
   (owner also allows uppercase input but is lowercased; repo `[A-Za-z0-9._-]{1,100}`),
   `path` normalized (no leading/trailing `/`, validated with the relPath rules
   from issue 05), empty path → `""` (repo root).
3. Ref validation: `/^[A-Za-z0-9._/-]{1,255}$/`, must not start with `-` or
   contain `..` — mirrors the git-injection defense in DESIGN 12.2.
4. Ambiguity rule for `parseAddRef`: a bare `owner/name` (no scheme, ≤ 1 slash)
   is a registry ref; anything with `github:`, `git+`, `://`, `git@`, or
   `file:` prefix routes to its kind; `owner/name@^1.2` splits range at the
   last `@` (range validated via `semver.validRange` or literal `latest`).
   Two-plus path segments without a scheme (`a/b/c`) → `E_USAGE` with hint
   "did you mean github:a/b/c#<ref>?".
5. Git URL forms accepted for `kind: "git"`: `git+https://…`, `https://…` (only
   when suffixed `.git` — otherwise E_USAGE hint to use github:), `ssh://git@…`,
   scp-style `git@host:owner/repo.git`. Path/ref supplied via
   `#<ref>` suffix and `::path=<relpath>` suffix (document exact syntax in
   `--help` text later; tests fix it here). Owner derivation for git URLs: the
   path segment before the repo name, lowercased; failure → E_USAGE.
6. Pure string module: no fs, no network, no other core imports besides errors
   (+ shared regex constants from schemas if exported there).

## Acceptance Criteria

- [ ] Table-driven tests: ≥ 25 accepted forms with exact expected structs; ≥ 20 rejected forms with `E_USAGE`/`E_CONFIG` and hint text asserted
- [ ] Round-trip: `specifierForLock(parseAddRef("github:Acme/Skills/tools/x"), sha)` yields `github:acme/Skills/tools/x#<sha>` string form with lowercase owner (repo case preserved)
- [ ] `owner/name@latest`, `owner/name@^1`, `owner/name@1.2.3` parse; `owner/name@junk` rejected
- [ ] scp-style and ssh URLs produce identical structs

## Validation

`npm test -- names` green.

## Dependencies

01, 02, 05.

## Non-goals

Actually resolving refs to SHAs (issues 12/19); manifest editing (issue 09).

## Design References

- DESIGN.md 6.1-6.2 (grammar), 12.2 (ref/URL injection defenses)
