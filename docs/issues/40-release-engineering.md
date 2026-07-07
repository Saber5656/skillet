# Title

Release engineering: hardened publish pipeline, provenance, repo security config

## Summary

Implement the release workflow (version → build → test → npm publish with
provenance) and the repository security configuration (Dependabot, CodeQL,
action pinning audit), completing the OSS release posture.

## Context

ADR-005/DESIGN 12.2 "skillet supply chain" row: exact-pinned deps, SHA-pinned
actions, provenance publishing. Secrets/org registration are user manual
actions (npm org `skillet`, `NPM_TOKEN` or OIDC trusted publisher setup) — the
workflow must be ready to run the moment the user completes them, and the
runbook must say exactly what the user does.

## Scope

- `.github/workflows/release.yml`, `.github/dependabot.yml`,
  `.github/workflows/codeql.yml`
- `docs/RELEASE.md` (runbook incl. user-manual-action checklist)
- `scripts/check-action-pins.ts` + CI step (fails on tag-referenced actions)
- CHANGELOG.md (keep-a-changelog skeleton) + release notes convention

## Detailed Requirements

1. `release.yml`: trigger `push: tags: v*`; jobs:
   (a) verify: full matrix test run (reuse ci.yml via `workflow_call`);
   (b) publish: needs verify; `environment: release`; steps: checkout
   (pinned SHA), setup-node with `registry-url`, `npm ci`, `npm run build`,
   assert tag == package.json version (script), `npm publish --provenance
   --access public`; `permissions: { id-token: write, contents: read }`;
   prefer OIDC trusted publishing — runbook documents the npmjs.com trusted
   publisher configuration (user manual) with `NPM_TOKEN` fallback via
   environment secret.
2. Version bump flow (documented, manual-friendly): edit version + CHANGELOG
   in a PR; tag after merge; workflow does the rest. No auto-bump bots in v1.
3. `dependabot.yml`: weekly, ecosystems `npm` (grouped minor/patch) and
   `github-actions`.
4. `codeql.yml`: default JS/TS analysis on PR + weekly schedule; pinned SHAs;
   `permissions: security-events: write`.
5. `check-action-pins.ts`: scans `.github/workflows/**` AND
   `index-repo/.github/workflows/**` for `uses:` not matching
   `@[0-9a-f]{40}`; wired into ci.yml; also asserts release workflow has no
   `pull_request_target`, no `secrets: inherit`.
6. `docs/RELEASE.md` runbook: prerequisites checklist explicitly marked
   **user manual actions** — register npm org `skillet` (fallback: rename to
   `skillets` per ADR-006 with the exact three files to edit), enable npm 2FA,
   configure trusted publisher or `NPM_TOKEN` env secret, enable GitHub
   settings (secret scanning + push protection, private vulnerability
   reporting, branch ruleset) — each with the exact UI path or `gh` command;
   then the per-release steps (5 numbered commands) and post-release
   verification (`npm view @skillet/cli version`, provenance badge check).
7. First-release gate: RELEASE.md includes the pre-1.0 checklist — all
   ISSUE_PLAN waves complete, e2e green 3× consecutive, security suite green,
   index repo instantiated (issue 36 runbook executed), README quickstart
   passes against the REAL registry once (documented manual verification, the
   only sanctioned real-network check).

## Acceptance Criteria

- [ ] `check-action-pins` passes on the repo and fails on a fixture with `@v4`
- [ ] Release workflow dry-runs green via `workflow_dispatch` with publish step in `--dry-run` mode behind an input flag
- [ ] Tag/version mismatch aborts before publish (unit-tested script)
- [ ] Dependabot + CodeQL configs lint (actionlint) and reference pinned SHAs
- [ ] RELEASE.md contains zero steps requiring the implementing agent to hold secrets; every credential step marked as user action

## Validation

CI green incl. new checks; `workflow_dispatch` dry-run evidence attached to
the implementing PR.

## Dependencies

01 (ci.yml), 36 (pin checker covers its workflows), 39 (CHANGELOG/README refs).

## Non-goals

Actual first publish and org registration (user actions); Homebrew tap,
signed releases, SLSA levels beyond npm provenance (v2).

## Design References

- DESIGN.md 12.2 (supply chain row), 15 (npm-name unknown), ADR-003/005/006
