# skillet — v1 Issue Plan

Status: Approved plan derived from docs/DESIGN.md (2026-07-07)
Issue drafts: `docs/issues/NN-*.md` (one file per issue; GitHub Issues are
derived artifacts of these files).

GitHub Issues [#1–#40](https://github.com/Saber5656/skillet/issues) were
created 2026-07-07 and correspond 1:1 to draft files `01-*.md` … `40-*.md`
(issue number == file number). If drafts and GitHub Issues ever disagree, the
draft files win; update them first, then sync the issue.

## v1 completion statement

v1 is complete when all 40 issues below are implemented and validated. At that
point the product delivers everything in DESIGN.md section 2 "v1 goals":
a cross-runtime (Claude Code, Codex) skill package manager with index-backed
search, semver resolution, lockfile reproducibility (`install --frozen`),
sha256 integrity verification, safe copy-based materialization, author
validation, and complete index-registry tooling + template — released as
`@skillet/cli` on npm with a hardened publish pipeline.

Three prerequisites are **user manual actions** outside the issues (tracked in
docs/RELEASE.md and docs/INDEX_REPO_SETUP.md): registering the npm org
`skillet` (or invoking the `skillets` fallback, ADR-006), instantiating the
`skillet-index` repository from the `index-repo/` template, and enabling
GitHub/npm security settings (2FA, trusted publishing, secret scanning).
Newly discovered implementation unknowns may add issues (see Known unknowns).

## Issue list in recommended execution order

| # | File | Title (short) | Wave |
|---|---|---|---|
| 01 | 01-project-scaffold.md | Project scaffold (build/test/lint/CI) | 0 |
| 02 | 02-error-taxonomy.md | Error taxonomy + exit codes | 0 |
| 03 | 03-config-and-paths.md | Config, paths, env, project root | 0 |
| 04 | 04-output-and-logging.md | Reporter + --json envelope | 0 |
| 05 | 05-schemas-and-models.md | zod models + JSON Schema export | 0 |
| 06 | 06-skillmd-parser-validator.md | SKILL.md parser/spec validator | 0 |
| 07 | 07-ref-grammar.md | Package/source ref grammar | 0 |
| 08 | 08-test-helpers.md | Sandbox, git fixtures, index server | 0 |
| 09 | 09-manifest-io.md | Manifest IO (+ shared jsonio) | 1 |
| 10 | 10-lockfile-io.md | Lockfile IO + diff | 1 |
| 11 | 11-index-client.md | Index fetch/cache/query | 1 |
| 12 | 12-git-mirror-fetch.md | spawn + git mirrors/fetch/refs | 1 |
| 13 | 13-git-tree-extraction.md | Tree enumeration + safe extraction | 1 |
| 14 | 14-integrity-hash.md | Canonical tree hash + vectors | 1 |
| 15 | 15-agent-adapters.md | Adapters (claude-code, codex) | 1 |
| 16 | 16-state-receipts.md | State receipts | 1 |
| 17 | 17-cli-entry.md | CLI entry, flags, error boundary | 2 |
| 18 | 18-cmd-init.md | `skillet init` | 2 |
| 19 | 19-resolver.md | Resolution/reconciliation planner | 2 |
| 20 | 20-staging-safety.md | Staging + safety filters/limits | 2 |
| 21 | 21-materializer.md | Atomic materialize/unmaterialize | 2 |
| 22 | 22-consent-prompt.md | Consent gate | 2 |
| 23 | 23-cmd-add.md | `skillet add` | 2 |
| 24 | 24-cmd-install.md | `skillet install` (+ --frozen) | 2 |
| 25 | 25-cmd-search.md | `skillet search` | 3 |
| 26 | 26-cmd-info.md | `skillet info` | 3 |
| 27 | 27-cmd-list.md | `skillet list` | 3 |
| 28 | 28-cmd-update.md | `skillet update` | 3 |
| 29 | 29-cmd-remove.md | `skillet remove` | 3 |
| 30 | 30-cmd-verify.md | `skillet verify` | 3 |
| 31 | 31-cmd-validate.md | `skillet validate` | 3 |
| 32 | 32-cmd-cache.md | `skillet cache dir/clean` | 3 |
| 33 | 33-indextools-validate.md | Index validation engine | 4 |
| 34 | 34-indextools-build.md | Aggregate builder + --check | 4 |
| 35 | 35-cmd-index.md | `skillet index validate/build` | 4 |
| 36 | 36-index-repo-template.md | index-repo/ template + runbook | 4 |
| 37 | 37-security-test-suite.md | Adversarial security suite | 5 |
| 38 | 38-e2e-user-journeys.md | E2E journey suite | 5 |
| 39 | 39-docs-readme-usage.md | README/USAGE/SECURITY/CONTRIBUTING | 5 |
| 40 | 40-release-engineering.md | Release pipeline + repo hardening | 5 |

## Dependency table

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01, 02 |
| 04 | 01, 02 |
| 05 | 01, 02, 03 |
| 06 | 01, 02 |
| 07 | 01, 02, 05 |
| 08 | 01, 02, 03 |
| 09 | 01, 02, 03, 05 |
| 10 | 01, 02, 03, 05, 09 |
| 11 | 01, 02, 03, 05, 08 |
| 12 | 01, 02, 03, 08 |
| 13 | 12 |
| 14 | 02, 13 (TreeEntry type only — may start in parallel) |
| 15 | 01, 02, 03 |
| 16 | 02, 03, 05, 09 |
| 17 | 02, 03, 04, 08 |
| 18 | 03, 04, 09, 15, 17 |
| 19 | 02, 05, 07, 10, 11, 12 (refResolver signature only) |
| 20 | 03, 06, 08, 12, 13, 14 |
| 21 | 03, 15, 16, 20 |
| 22 | 02, 04, 16 |
| 23 | 07, 09, 10, 11, 12, 15, 16, 17, 19, 20, 21, 22 |
| 24 | 09, 10, 11, 12, 15, 16, 17, 19, 20, 21, 22 |
| 25 | 03, 04, 11, 17 |
| 26 | 03, 04, 07, 11, 17 |
| 27 | 03, 04, 09, 10, 15, 16, 17 |
| 28 | 10, 11, 12, 17, 19, 20, 21, 22, 23 (shared pipeline helpers) |
| 29 | 09, 10, 16, 17, 21, 22 |
| 30 | 10, 14, 16, 17 |
| 31 | 06, 14, 17, 20 |
| 32 | 03, 04, 12, 17, 20 |
| 33 | 05, 06, 08, 12, 13, 14, 20 |
| 34 | 05, 09, 33 (fixtures) |
| 35 | 17, 33, 34 |
| 36 | 31, 35 |
| 37 | 08, 12, 13, 20, 21, 23, 24, 29 |
| 38 | 08, 18, 23–32, 36 |
| 39 | 18, 23–32, 35, 36 |
| 40 | 01, 36, 39 |

Within a wave, issues sharing no edge in this table can proceed in parallel
(e.g. 09/11/12/15/16 concurrently; 25/26/27/30/31/32 concurrently).

## Implementation waves

| Wave | Issues | Outcome / gate to next wave |
|---|---|---|
| 0 — Foundations | 01–08 | Repo builds/tests on 6-cell CI matrix; every format, error, path, and spec rule has a tested home |
| 1 — Data & sources | 09–16 | All persistence + git + integrity + adapter primitives green; no CLI yet |
| 2 — Core flows | 17–24 | `init/add/install --frozen` work end-to-end in sandboxes: the product promise (reproducible team installs) is demonstrable |
| 3 — Full command surface | 25–32 | Every DESIGN 14 command implemented; CLI surface frozen |
| 4 — Registry ecosystem | 33–36 | Index tooling + copy-ready index repo template; author path complete |
| 5 — Hardening & release | 37–40 | Threat-table coverage proven, journeys green 3× on matrix, docs synced, release pipeline dry-run green → tag v0.1.0 |

## Coverage table (DESIGN.md section → issues)

| DESIGN section | Covered by issues |
|---|---|
| 1 Positioning | 39 |
| 2 Goals/non-goals | whole plan (this file) |
| 4 Primary workflows | 18, 23, 24, 28, 38 |
| 5 Architecture/module layout | 01, 02, 03, 04, 05, 17 |
| 6.1 Package identity | 05, 07, 19, 33 |
| 6.2 Dependency specifiers | 05, 07, 19, 23 |
| 7.1 Manifest | 05, 09, 18 |
| 7.2 Lockfile | 05, 10, 24, 30 |
| 7.3 State | 05, 16, 21, 27 |
| 7.4 Config/env | 03, 05, 32 |
| 7.5 Index entry | 05, 33 |
| 7.6 Aggregate index | 05, 11, 34 |
| 8 Canonical tree hash + limits | 13, 14, 20, 31, 37 |
| 9 Resolution semantics | 19, 23, 24, 28, 29 |
| 10.1 Pipeline state machine | 20, 22, 23, 24, 28 |
| 10.2 Git fetch | 12, 13, 32 |
| 10.3 Materialization | 21, 23, 24, 29 |
| 11 Index registry | 33, 34, 35, 36 |
| 12.1–12.3 Security model | 03, 05, 11, 12, 13, 20, 21, 37, 39, 40 |
| 12.4 Consent UX | 22, 23, 24, 28 |
| 13 Adapters | 15, 18, 21 |
| 14 CLI spec | 02, 04, 17, 18, 23–32, 35, 39 |
| 15 Deferred/unknowns | this file (below) |
| 16 Testing strategy | 08, 37, 38 + per-issue Validation sections |

Every normative DESIGN section maps to at least one issue; no v1 behavior
lives only in prose.

## Validation strategy (whole product)

1. **Per-module unit tests** (issues 02–16, 19–22, 33–34): every format,
   algorithm, and safety rule tested in isolation with sandboxes and fixture
   repos; integrity test vectors are frozen as normative artifacts (14).
2. **Per-command tests** (17, 18, 23–32, 35): each command's flags, outputs
   (human + `--json` envelope), exit codes, and rollback behavior through the
   real CLI in sandboxes.
3. **Adversarial security suite** (37): one executable scenario per DESIGN
   12.2 threat row, with a meta-test that fails CI if coverage regresses.
4. **E2E journeys** (38): the five DESIGN 4 workflows plus offline and
   json-contract journeys, spawning the built binary only; journey 1 proves
   byte-identical team reproduction — the core promise.
5. **Docs-sync guard** (39): CLI surface and USAGE.md cannot drift (test).
6. **Cross-platform determinism**: 6-cell CI matrix (3 OS × Node 20/22) from
   issue 01 onward; byte-stability tests for lockfile/aggregate serialization.
7. **Release gate** (40): tag-driven pipeline re-runs the full matrix, then
   provenance-publishes; RELEASE.md pre-1.0 checklist requires 3× consecutive
   green e2e and one documented manual quickstart run against the real
   registry.

## Deferred v2 items

From DESIGN 15: `skillet publish` (index-PR automation), index/package signing
(sigstore), advisory/yank flow beyond `deprecated`, flat-name aliases,
prompt-injection content heuristics, additional adapters (Cursor, OpenCode,
Copilot CLI, Gemini CLI), sharded/CDN-hosted index, parallel fetch/staging,
`skillet new` scaffolding, private indexes, popularity/download metrics,
Homebrew tap distribution, `info --readme`.

## Known unknowns (may create additional issues)

| Unknown | Watch point | Likely follow-up |
|---|---|---|
| npm org `skillet` availability | User registration attempt (RELEASE.md step 1) | Rename issue applying the ADR-006 fallback (`skillets`) across 3 files |
| Host SHA-fetch variance (GitLab/Codeberg self-hosted) | Issue 12 fallback-chain telemetry in real use | Extra fallback strategy issue |
| Aggregate index growth | Registry > ~2k packages or > 5 MiB | Sharding + CDN issue set (v2 pull-forward) |
| `skills-ref` parity | Spec evolution at agentskills.io | Conformance-corpus test issue |
| Codex layout changes (nested skills, AGENTS.md interplay) | Codex release notes | Adapter revision issue |
| Windows exec-bit semantics in verify (`ok-execbits` rule) | Issue 30 field feedback | Refined per-file class recording (schema v2) |
| Case-insensitive FS collisions beyond skill trees (owner dirs in cache) | Issue 32/12 on macOS/Windows CI | Cache layout hashing tweak |
