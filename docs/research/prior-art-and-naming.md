# Research: Prior Art and Naming

- Date verified: 2026-07-07
- Status: informs positioning (DESIGN.md "Positioning") and distribution naming (ADR-006)

## Direct prior art

### vercel-labs/skills (`npx skills`)

Source: https://github.com/vercel-labs/skills (README fetched 2026-07-07)

The closest existing tool. Feature matrix as verified:

| Capability | `npx skills` | skillet v1 (planned) |
|---|---|---|
| Cross-agent install | Yes (72+ agents) | Yes (Claude Code, Codex; adapter interface for more) |
| Sources | GitHub shorthand/URL, GitLab, any git URL, local path | Registry index + git direct + local path |
| Scopes | project / global | project / user |
| Install method | symlink (default) or copy | copy (atomic, receipt-tracked) |
| Search | `skills find` (keyword, `--owner`; backed by skills.sh popularity directory) | curated index search (offline after fetch) |
| Version pinning | No (installs current HEAD) | Yes (semver versions -> pinned commit) |
| Lockfile / reproducible install | No | Yes (`skillet.lock`, `install --frozen`) |
| Integrity verification | No | Yes (sha256 canonical tree hash, `skillet verify`) |
| Curation / anti-typosquat | No (any repo) | Index CI enforces owner binding + validation |
| Update semantics | "update to latest" | range-constrained, lock-diffed updates |

Gap skillet fills: **npm-grade reproducibility and supply-chain rigor** (versions,
lockfile, integrity, curated namespace). This matches the repository README's
one-line charter (検索・インストール・バージョン管理・lockfile).

### dafrick/skillet-cli (name-adjacent, domain-adjacent)

- npm: `skillet-cli` (active; created 2026-06-01, publishing through 2026-07),
  `@skillet-cli/core` ("Ship your skill as an npm package with a complete CLI for
  any agent environment"), `create-skillet` ("Interactive wizard to package a skill
  for any AI agent"). Repo: https://github.com/dafrick/skillet-cli
- Different product shape (packages a single skill as an npm package) but shares the
  "skillet" brand in the same broad space. Coexistence requires clear positioning
  and a non-colliding npm name.

### Other ecosystem tools/directories

- `add-skill` (https://add-skill.org) — installer for OpenCode/Claude Code/Codex/Cursor.
- skills.sh (Vercel) — popularity directory backing `skills find`.
- skills-hub.ai, mdskills.ai, agensi.io — SKILL.md directories/marketplaces.
- Claude Code plugin marketplaces — first-party, Claude-Code-only distribution.
- `skills-ref` (npm) — official Agent Skills reference validation library.

None of these provide versioned, lock-file-reproducible, integrity-verified installs.

## npm namespace facts (verified against registry.npmjs.org, 2026-07-07)

| Name | Status |
|---|---|
| `skillet` | Taken (2011, "Identify duplicate javascript strings", effectively abandoned; owns bin `skillet`) |
| `skillet-cli` | Taken, active (dafrick) |
| `@skillet-cli/*`, `create-skillet` | Taken, active (dafrick) |
| `@skillet/*` scope | Zero packages published under the scope; org registration status unknowable without attempting registration |
| `skillets`, `skilletjs`, `skillet-kit`, `skillet-manager`, `skillet-pm`, `skilletpm`, `cast-iron`, `panfry` | Free at verification time |

Decision (user-approved 2026-07-07): distribute as **`@skillet/cli`** with bin
`skillet`; fallback **`skillets`** if the npm org "skillet" cannot be registered.
Org registration is a manual user action. See ADR-006.

Note: the abandoned 2011 `skillet` package also declares a `skillet` bin; a global
bin collision occurs only if a user installs both, which is acceptable residual risk.

## Non-npm name collisions

- Palo Alto Networks "Skillet" (PAN-OS configuration templates, skilletlib) —
  unrelated domain, older; low confusion risk.

## Sources

- [vercel-labs/skills](https://github.com/vercel-labs/skills)
- [dafrick/skillet-cli](https://github.com/dafrick/skillet-cli)
- [add-skill](https://add-skill.org/)
- [OpenAI Developers: Agent Skills - Codex](https://developers.openai.com/codex/skills)
- npm registry API queries (`registry.npmjs.org`), 2026-07-07
