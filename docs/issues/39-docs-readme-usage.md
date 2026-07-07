# Title

User documentation: README, USAGE, SECURITY.md, CONTRIBUTING for the CLI repo

## Summary

Write the public-facing documentation for the skillet repository: a README
that sells the differentiator in the first screen, a complete USAGE reference
generated-from/verified-against the real CLI, the security policy, and
contributor onboarding.

## Context

The repo is an OSS release (task charter); docs are part of v1 completeness.
Positioning and honest security claims come from DESIGN 1 and 12 / ADR-005
(no overclaiming: signing is v2 and SECURITY.md must say so). The existing
one-line Japanese README is replaced by an English README that keeps the
original Japanese one-liner as a tagline translation note.

## Scope

- `README.md` (rewrite), `docs/USAGE.md`, `SECURITY.md`, `CONTRIBUTING.md`
- `tests/docs/usage-sync.test.ts` (docs-vs-CLI drift guard)

## Detailed Requirements

1. `README.md` sections, in order: what/why (the reproducibility gap vs
   `npx skills` — factual one-paragraph comparison linking
   docs/research/prior-art-and-naming.md); install
   (`npm i -g @skillet/cli`, npx form); 60-second quickstart (init → search →
   add → commit lock → teammate `install --frozen`); supported runtimes table;
   security posture summary (5 bullets from ADR-005 + link to SECURITY.md);
   project status badge-honesty (v1, index registry link); license.
2. `docs/USAGE.md`: every command with synopsis, flags, examples, exit codes,
   and `--json` output shape — content must match `--help` texts; the sync
   test asserts every command and flag in the CLI appears in USAGE.md and vice
   versa (parse commander program vs a structured block in the doc:
   maintain a fenced `<!-- cli-surface -->` JSON block the test regenerates
   and diffs).
3. `SECURITY.md`: supported versions; report channel (GitHub private
   vulnerability reporting); the threat model summary table (from DESIGN 12.2)
   with honest residual risks (index root-of-trust, no signing yet, no content
   scanning); disclosure timeline commitment (90 days).
4. `CONTRIBUTING.md`: dev setup (`npm ci`, build/test), module map pointer to
   DESIGN 5, issue workflow (issues derive from docs/issues — PRs that change
   behavior must update DESIGN.md), the dependency-budget rule (ADR-003), and
   commit/PR conventions (English, conventional commits).
5. All docs use the final npm name; a single-source constant note: the name
   appears in README/USAGE only via literal strings listed in the issue-01
   fallback checklist (grep-able `@skillet/cli`).

## Acceptance Criteria

- [ ] usage-sync test green (CLI surface == USAGE.md block)
- [ ] README quickstart commands execute verbatim against a fixture registry in a sandbox (scripted test, not manual)
- [ ] SECURITY.md contains the residual-risk sentences for: index compromise, no signing (v2), no content scanning (v2)
- [ ] No doc claims a feature the CLI lacks (reviewer checklist + the sync test)
- [ ] Original Japanese one-liner preserved in README (tagline)

## Validation

`npm test -- docs` green; markdownlint (add as devDependency is NOT allowed —
use a plain assertion script for heading structure instead).

## Dependencies

All command issues (surface must be final): 18, 23-32, 35; 36 (registry links).

## Non-goals

Website/docs-site (v2); translated docs beyond the tagline (v2); CHANGELOG
automation (issue 40).

## Design References

- DESIGN.md 1 (positioning), 12 (security honesty), 14 (surface), ADR-005/006
