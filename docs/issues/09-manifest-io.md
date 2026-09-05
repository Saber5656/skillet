# Title

core/manifest.ts: skillet.json read, edit, and atomic write

## Summary

Implement reading, validating, editing (add/remove entries), and atomically
writing the manifest for both scopes, with deterministic serialization.

## Context

DESIGN.md 7.1 defines `skillet.json`. Commands `init/add/remove` mutate it;
everything else reads it. Writes must be atomic (tmp file + rename) and
byte-stable so diffs stay minimal in team repos.

## Scope

- `src/core/manifest.ts`
- `tests/core/manifest.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export function readManifest(path: string): Manifest;            // E_CONFIG if invalid; E_CONFIG w/ hint "run skillet init" if missing
   export function readManifestIfExists(path: string): Manifest | null;
   export function writeManifest(path: string, m: Manifest): void;  // atomic, stable
   export function newManifest(agents: string[]): Manifest;         // includes $schema URL from DESIGN 7.1
   export function upsertSkill(m: Manifest, id: string, spec: DependencySpecifier): Manifest; // pure
   export function removeSkill(m: Manifest, id: string): Manifest;  // pure; E_UNKNOWN_PACKAGE if absent
   ```
2. Stable serialization (shared util `src/core/jsonio.ts`, also used by issue 10):
   `writeJsonStable(path, value)` — recursively sorted keys EXCEPT: the top-level
   key order for manifest is fixed `["$schema","agents","skills"]`; `skills`
   keys sorted; 2-space indent; trailing newline; `\n` line endings; atomic via
   `write to <path>.tmp-<pid>` then `fs.renameSync`.
3. Validation through `parseManifest` (issue 05). Unknown adapter ids are NOT
   validated here (registry knowledge lives in adapters; resolver/commands
   check) — document this boundary in a code comment.
4. `readManifest` must reject a manifest whose `skills` keys fail PackageId
   rules with the offending key named in the message.
5. No caching; each call re-reads the file (commands are short-lived).

## Acceptance Criteria

- [ ] Round-trip: read(write(m)) deep-equals m; write twice → identical bytes
- [ ] upsert preserves other entries; remove of missing id throws `E_UNKNOWN_PACKAGE`
- [ ] Missing file: `readManifest` throws with init hint; `readManifestIfExists` returns null
- [ ] Atomicity: no partial file remains when the target dir is read-only (write fails cleanly, original intact)
- [ ] Key order in output exactly `$schema, agents, skills` with sorted skills

## Validation

`npm test -- manifest` green on CI matrix.

## Dependencies

01, 02, 03, 05.

## Non-goals

Resolution or range semantics (issue 19); lockfile (issue 10); init command
UX (issue 18).

## Design References

- DESIGN.md 7.1 (format), section 9 (which operations mutate the manifest)
