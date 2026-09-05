# Title

index-repo/: complete, copy-ready template for the skillet-index repository

## Summary

Produce the full content of the separate `skillet-index` repository as a
template directory in this repo (`index-repo/`), including CI workflows,
contribution docs, PR template, and seed entries — so instantiating the real
registry is a copy + push, with no authoring left to do.

## Context

ADR-002/ADR-004: the index is a separate GitHub repository
(`Saber5656/skillet-index`), populated via PRs validated by
`skillet index validate` and aggregated by `skillet index build`. Creating the
actual GitHub repository is a user manual action (account-level operation);
this issue makes that action mechanical.

## Scope

- `index-repo/` directory in this repository (template), containing:
  `README.md`, `CONTRIBUTING.md`, `SECURITY.md`,
  `.github/workflows/validate.yml`, `.github/workflows/build.yml`,
  `.github/PULL_REQUEST_TEMPLATE.md`, `registry/.gitkeep`,
  seed entry `registry/saber5656/hello-skillet.json` (+ its source skill under
  this repo's `examples/hello-skillet/` so the seed points at a real, owned
  source), and a root `index.json` built from the seed
- `docs/INDEX_REPO_SETUP.md`: numbered instantiation runbook (create repo →
  copy template → push → branch protection checklist referencing the user's
  standard ruleset practice)

## Detailed Requirements

1. `validate.yml` (PR trigger, paths `registry/**`): checkout with
   `fetch-depth: 0`; setup-node 20; `npx @skillet/cli@latest index validate
   --changed-since origin/main --strict` and `npx @skillet/cli@latest index
   build --check`; actions pinned by commit SHA; `permissions: contents: read`.
2. `build.yml` (push to main, paths `registry/**`): rebuild `index.json`,
   commit with `github-actions[bot]` identity ONLY when changed;
   `permissions: contents: write`; concurrency group to serialize builds;
   commit message `chore: rebuild index.json`.
3. `CONTRIBUTING.md` documents the exact author flow (ADR-004): validate
   locally (`skillet validate --print-integrity`), craft
   `registry/<owner>/<name>.json` (annotated example with every field of
   DESIGN 7.5), the four PR rules (one package per PR; owner must match source
   URL owner; published versions immutable; deprecation instead of deletion),
   and what CI will reject.
4. `SECURITY.md` (index repo): report channel placeholder
   (GitHub private vulnerability reporting), takedown/deprecation policy for
   malicious packages: maintainers set `deprecated` + remove versions ONLY in
   confirmed-malware cases (the one sanctioned immutability exception,
   requiring a security advisory note in the PR).
5. `examples/hello-skillet/`: minimal valid skill (SKILL.md name
   `hello-skillet`, description, one `references/USAGE.md`) that also serves
   as a living fixture for docs; seed entry pins a commit of THIS repo once
   pushed — the runbook explains computing it post-push (template ships with
   `REPLACE_ME_COMMIT`/`REPLACE_ME_INTEGRITY` placeholders and the exact
   commands to fill them).
6. Everything in `index-repo/` must pass `skillet index validate --no-network`
   structurally as checked by a test in THIS repo (placeholders included via a
   test-time substitution helper).

## Acceptance Criteria

- [ ] `skillet index validate --no-network index-repo` → zero findings after placeholder substitution (automated test)
- [ ] `skillet index build --check` passes on the template (with substitution)
- [ ] Workflows lint clean (actionlint in this repo's CI for `index-repo/**`)
- [ ] Runbook executable top-to-bottom by a non-expert: every command literal, no "figure out X" steps
- [ ] Seed skill passes `skillet validate`

## Validation

Repo tests above; manual read-through of the runbook against a scratch GitHub
account is documented as a release-checklist item (issue 40), not this issue.

## Dependencies

31, 35 (commands used inside workflows/tests).

## Non-goals

Creating the real GitHub repository or registering anything (user manual
action per runbook); index hosting beyond raw.githubusercontent (v2).

## Design References

- DESIGN.md section 11 (layout/flows), 7.5-7.6, ADR-002, ADR-004
