# Title

core/skillmd.ts: SKILL.md frontmatter parser and Agent Skills spec validator

## Summary

Parse `SKILL.md` (YAML frontmatter + body) and validate a skill directory
against the Agent Skills specification, producing the metadata object used by
staging, consent, validate command, and index CI.

## Context

The exact spec constraints are recorded in
`docs/research/agent-skills-spec.md` (verified 2026-07-07): required `name`
(1-64, `[a-z0-9-]`, no edge/consecutive hyphens, == directory basename) and
`description` (1-1024); optional `license`, `compatibility` (1-500),
`metadata` (string→string map), `allowed-tools` (space-separated string).
Unknown fields must warn, not fail (runtimes extend the spec).

## Scope

- `src/core/skillmd.ts`
- `tests/core/skillmd.test.ts` + fixtures under `tests/fixtures/skillmd/`

## Detailed Requirements

1. Export:
   ```ts
   export interface SkillMeta {
     name: string; description: string;
     license?: string; compatibility?: string;
     metadata?: Record<string, string>; allowedTools?: string;
     warnings: string[];              // unknown fields, oversize body, etc.
   }
   export function parseSkillMd(content: string, file: string): SkillMeta; // throws E_SPEC_INVALID
   export interface SkillDirInfo extends SkillMeta {
     dir: string; fileCount: number; totalBytes: number;
     executables: string[];           // relative paths with owner-exec bit / mode 100755
   }
   export function inspectSkillDir(dir: string): Promise<SkillDirInfo>; // throws E_SPEC_INVALID
   ```
2. Frontmatter extraction: content must start with `---\n`; frontmatter ends at
   the next line that is exactly `---`; parse with `yaml` package
   (`strict: true`, no custom tags, `maxAliasCount: 10` to block YAML bombs).
   Body is not parsed (only counted for the < 5000-token guidance warning:
   warn when body > 48 KiB).
3. Field validation exactly per the research doc table. `metadata` values must
   be strings (numbers/booleans → `E_SPEC_INVALID` with a hint to quote them —
   matches spec "map from string keys to string values").
4. Unknown top-level frontmatter keys → warning string
   `unknown frontmatter field "<key>" (runtime extension?) — passed through`.
5. `inspectSkillDir`: requires `SKILL.md` directly in `dir`; validates
   `meta.name === basename(dir)`; walks files (`node:fs` `readdir` recursive,
   `lstat` — symlinks are counted and reported by callers via stage filters,
   not here); collects `fileCount`, `totalBytes`, `executables` (POSIX exec bit;
   on Windows report `[]`).
6. No limits enforcement here (stage/issue 20 owns limits) — this module is
   spec-only, so index CI and client share identical spec semantics.

## Acceptance Criteria

- [ ] Fixtures: minimal valid; full-featured valid; missing name; name/dir mismatch; uppercase name; `a--b`; 1025-char description; 501-char compatibility; numeric metadata value; unknown field (warns, passes); frontmatter missing; YAML alias bomb (fails fast)
- [ ] `allowed-tools` string preserved verbatim (no tokenization in v1)
- [ ] Windows CI: `executables` returns `[]` without error
- [ ] Error messages include the file path and the failing field name

## Validation

`npm test -- skillmd` green on the CI matrix.

## Dependencies

01, 02 (05 for shared regex helpers if convenient, else self-contained).

## Non-goals

Tree safety filters and size limits (issue 20); `skillet validate` CLI (issue 31).

## Design References

- docs/research/agent-skills-spec.md (normative constraints)
- DESIGN.md 6.1 (name reuse), 12.4 (consent uses executables/allowed-tools)
