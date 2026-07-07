# Title

core/schemas.ts: zod models for manifest, lockfile, state, config, index + JSON Schema export

## Summary

Implement the single source of truth for every file format skillet reads or
writes, as zod schemas with inferred TypeScript types, plus a build step that
exports JSON Schemas to `schemas/` for editor `$schema` support.

## Context

DESIGN.md section 7 specifies five formats: manifest (7.1), lockfile (7.2),
state (7.3), user config (7.4), index entry (7.5) and aggregate index (7.6).
All later modules must import these types instead of re-declaring shapes.

## Scope

- `src/core/schemas.ts`
- `scripts/generate-json-schemas.ts` (dev script) and generated `schemas/*.json`
- `tests/core/schemas.test.ts`
- Refactor `core/config.ts` `TODO(issue-05)` marker to use `UserConfigSchema`

## Detailed Requirements

1. Implement and export (schema + `z.infer` type for each):
   - `PackageIdSchema`: string matching
     `/^[a-z0-9](?:[a-z0-9-]{0,37}[a-z0-9])?\/[a-z0-9]+(?:-[a-z0-9]+)*$/` AND the
     skill segment ≤ 64 chars (superRefine; reuse the exact name rules from
     DESIGN 6.1: owner 1-39 `[a-z0-9-]` no edge hyphen; skill per Agent Skills
     spec incl. no consecutive hyphens).
   - `DependencySpecifierSchema`: union of
     (a) semver range string (validated via `semver.validRange`) or `"latest"`,
     (b) `github:` shorthand string (regex per DESIGN 6.2, requires `#ref`),
     (c) `file:` string, (d) object `{ git: httpsOrSshUrl, path: relPath, ref: string }`.
   - `ManifestSchema` (7.1): `$schema?` string, `agents` non-empty string array,
     `skills` record<PackageId, DependencySpecifier> (default `{}`).
   - `ResolvedSourceSchema`: discriminated union on `type`:
     `index`/`git` → `{ url, commit: /^[0-9a-f]{40}$/, path: relPath }`,
     `path` → `{ path: string }`.
   - `LockfileSchema` (7.2): `lockfileVersion: literal 1`, `skills`
     record<PackageId, { version, resolved, integrity: /^sha256-[A-Za-z0-9+/]+=*$/,
     portable: boolean }>.
   - `StateSchema` (7.3): `stateVersion: literal 1`, `installs` array of
     `{ package, version, integrity, agent, dir, installedAt: ISO datetime }`.
   - `UserConfigSchema` (7.4): all optional `{ indexUrl: httpsUrl,
     defaultAgents: string[], cacheDir: string }`, `.passthrough()` replaced by
     strip (unknown keys ignored).
   - `IndexEntrySchema` (7.5) incl. `versions` array item
     `{ version: semver, commit, path: relPath, integrity, publishedAt }`,
     `deprecated: null | { reason, replacement: PackageId | null }`,
     `source: { type: literal "git", url: httpsUrl }`, `maintainers`, `keywords`
     (≤ 10 items, each `/^[a-z0-9-]{1,32}$/`), `description` 1-1024.
   - `AggregateIndexSchema` (7.6).
   - `relPath` helper schema: non-empty, forward slashes only, no `\`, no NUL,
     no segment `.` or `..`, no leading `/`, ≤ 512 bytes.
2. `scripts/generate-json-schemas.ts`: uses zod v4 native `z.toJSONSchema` to
   emit `schemas/skillet.schema.json`, `schemas/skillet-lock.schema.json`,
   `schemas/index-entry.schema.json`; wire `npm run build` to run it before tsup.
   Generated files are committed.
3. Parse helpers: `parseManifest(json: unknown, file: string): Manifest` etc.,
   converting zod errors to `SkilletError("E_CONFIG" | "E_SPEC_INVALID", msg
   with file path + first 3 zod issues, hint)` — manifest/lock/state/config use
   `E_CONFIG`; index data uses `E_SPEC_INVALID`.

## Acceptance Criteria

- [ ] Round-trip: DESIGN 7.1/7.2/7.3/7.5 examples parse successfully (fixtures copied verbatim from DESIGN.md)
- [ ] Rejections with precise messages: bad package id (`Owner/Name`, `a//b`, 65-char name, `a--b` skill), non-40-hex commit, `..` in path, http:// index source, 11 keywords
- [ ] `github:acme/skills/tools/x` (missing `#ref`) rejected; with `#ref` accepted
- [ ] `schemas/*.json` regenerate deterministically (build twice → no diff)
- [ ] config.ts TODO removed; its tests still green

## Validation

`npm test -- schemas config` green; `npm run build` produces committed schemas
with no diff.

## Dependencies

01, 02, 03.

## Non-goals

Ref-string parsing into structured objects (issue 07); stable serialization
(issues 09/10); semver resolution (issue 19).

## Design References

- DESIGN.md section 7 (all formats), 6.1-6.2 (identity/specifier grammar)
