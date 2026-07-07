# Research: Agent Skills Specification

- Date verified: 2026-07-07
- Primary source: https://agentskills.io/specification (fetched as https://agentskills.io/specification.md)
- Status: normative input for skillet's SKILL.md validation and packaging rules

## Summary

A skill is a directory containing at minimum a `SKILL.md` file. `SKILL.md` is YAML
frontmatter followed by a Markdown body. Optional conventional subdirectories:
`scripts/`, `references/`, `assets/`. Any additional files are allowed.

## Frontmatter fields (normative)

| Field | Required | Constraints |
|---|---|---|
| `name` | Yes | 1-64 chars; lowercase `a-z`, `0-9`, hyphens only; must not start or end with a hyphen; no consecutive hyphens (`--`); **must match the parent directory name** |
| `description` | Yes | 1-1024 chars, non-empty |
| `license` | No | Free-form short string (license name or bundled license file reference) |
| `compatibility` | No | 1-500 chars if present; environment requirements |
| `metadata` | No | Map of string keys to string values; conventionally carries `author`, `version` |
| `allowed-tools` | No | Space-separated string of pre-approved tools. Experimental; support varies by runtime |

Implementation-specific extension fields exist outside the core spec (Claude Code
adds e.g. `context`, `model`, `user-invocable`). A validator must not reject unknown
fields; it should warn at most.

## Consequences for skillet

1. **Package name segment must satisfy the spec `name` rules** — skillet reuses the
   exact regex constraints for the skill segment of its package identity.
2. **Directory name == `name` is a hard runtime constraint.** Two different packages
   whose skill `name` collides cannot be materialized into the same agent scope.
   skillet must detect and reject basename collisions at resolution time.
3. **`version` is not a first-class spec field.** It conventionally lives in
   `metadata.version` as a string. Therefore version authority for skillet cannot be
   the SKILL.md file alone; the index registry carries the authoritative
   version -> commit mapping, and `metadata.version` (when present) is cross-checked.
4. **`scripts/` may contain executable code.** Installation must surface the presence
   of executables to the user (consent prompt) but never execute anything itself.
5. **Reference validator exists**: `skills-ref` (npm, "Reference library for Agent
   Skills", maintained under the agentskills org,
   https://github.com/agentskills/agentskills). skillet v1 implements its own
   validator (exact rules above, small surface) to keep the dependency tree minimal;
   parity with `skills-ref` is tracked as a known unknown / test concern.

## Sources

- [Agent Skills Specification](https://agentskills.io/specification)
- [agentskills/agentskills reference repo (skills-ref)](https://github.com/agentskills/agentskills)
- [Claude Code skills documentation](https://code.claude.com/docs/en/skills) (confirms Claude Code follows the open standard)
