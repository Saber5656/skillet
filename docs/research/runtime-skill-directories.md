# Research: Runtime Skill Directories (Claude Code, Codex)

- Date verified: 2026-07-07
- Status: normative input for skillet's agent adapters

## Claude Code

Source: https://code.claude.com/docs/en/skills (fetched as markdown)

| Scope | Location | Notes |
|---|---|---|
| Personal (user) | `~/.claude/skills/<skill-name>/SKILL.md` | Available across all projects |
| Project | `<project>/.claude/skills/<skill-name>/SKILL.md` | Committed with the project |
| Plugins | plugin-provided | Out of scope for skillet v1 (plugin marketplace is a separate distribution channel) |

Additional confirmed facts:

- Claude Code skills follow the Agent Skills open standard (agentskills.io).
- The directory name becomes the `/command` name; directory name must equal
  frontmatter `name`.
- Custom slash commands (`.claude/commands/*.md`) are a legacy sibling mechanism;
  skillet does not manage them.
- Claude Code extends the spec with extra frontmatter (`context`, `model`,
  `user-invocable`, dynamic context injection). skillet must pass these through
  untouched and must not reject them.

## Codex CLI

Sources: https://developers.openai.com/codex/skills; multiple independent 2025-2026
writeups (see below).

| Scope | Location | Notes |
|---|---|---|
| Personal (user) | `~/.codex/skills/<skill-name>/` | Any folder there is treated as a skill |
| Project | `<project>/.codex/skills/<skill-name>/` | Project-scoped skills |

Additional confirmed facts:

- Codex consumes the same SKILL.md format (name + description required,
  progressive disclosure of body content).
- Skills activate automatically by description match or explicitly via `$`
  mention / `/skills`.

## Consequences for skillet

1. Both v1 runtimes support both `user` and `project` scopes with the identical
   layout pattern `<base>/skills/<skill-name>/`. The adapter interface can be a
   simple base-directory mapping per scope.
2. Runtime detection heuristic: adapter is "detected" if its base directory exists
   (`~/.claude` / `~/.codex` for user scope; presence is not required for project
   scope, where targeting is driven by the project manifest).
3. Installed payload must be a plain directory tree (no symlinks): both runtimes
   read files directly; copies are the most portable materialization (Windows
   symlink privileges, iCloud/Dropbox-synced home directories).

## Sources

- [Claude Code: Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [OpenAI Developers: Agent Skills - Codex](https://developers.openai.com/codex/skills)
- [Simon Willison: OpenAI are quietly adopting skills](https://simonwillison.net/2025/Dec/12/openai-skills/)
- [blog.fsck.com: Skills in OpenAI Codex](https://blog.fsck.com/2025/12/19/codex-skills/)
