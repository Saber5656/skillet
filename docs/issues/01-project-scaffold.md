# Title

Project scaffold: package.json, TypeScript, build, test, lint, CI skeleton

## Summary

Create the buildable, testable, lintable empty shell of the `@skillet/cli`
package so every later issue only adds modules and tests.

## Context

The repository currently contains only `README.md` and `docs/`. ADR-003 fixes
the stack: TypeScript, Node >= 20, ESM-only, tsup bundling, vitest, npm
distribution as `@skillet/cli` with bin `skillet` (fallback name `skillets` per
ADR-006 — a single `package.json` string swap decided at publish time).

## Scope

- `package.json`, `tsconfig.json`, `tsup.config.ts`, `vitest.config.ts`,
  `eslint.config.js`, `.prettierrc.json`, `.gitignore`, `.editorconfig`
- `LICENSE` (MIT, copyright holder `Saber5656`) — flagged for user confirmation
  in the PR description, not silently final
- `src/cli/index.ts` printing name+version (placeholder), `src/version.ts`
- `.github/workflows/ci.yml`
- One smoke test

## Detailed Requirements

1. `package.json`:
   - `"name": "@skillet/cli"`, `"version": "0.0.0"`, `"type": "module"`,
     `"license": "MIT"`, `"engines": {"node": ">=20"}`,
     `"bin": {"skillet": "./dist/cli.js"}`, `"files": ["dist", "schemas"]`.
   - Runtime deps exactly: `commander`, `zod` (v4+, native JSON Schema export),
     `yaml`, `semver`, `picocolors`. Dev deps: `typescript`, `tsup`, `vitest`,
     `@types/node`, `eslint` (+ typescript-eslint), `prettier`.
   - Scripts: `build` (tsup), `test` (vitest run), `test:watch`, `lint`
     (eslint + prettier --check), `format`, `typecheck` (tsc --noEmit).
   - Commit `package-lock.json` (exact pinning is a security control, ADR-005).
2. `tsconfig.json`: `strict: true`, `module/moduleResolution: NodeNext`,
   `target: ES2022`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.
3. tsup: entry `src/cli/index.ts` → `dist/cli.js` (ESM, node20 target,
   shebang `#!/usr/bin/env node` preserved), plus `src/index.ts` library entry
   exporting nothing yet (placeholder for indextools reuse).
4. `src/version.ts` reads the version from `package.json` at build time
   (tsup `define` or JSON import with assertion) — no runtime `require`.
5. `src/cli/index.ts`: prints `skillet <version>` and exits 0 (real CLI arrives
   in issue 17). Must run via `node dist/cli.js` after `npm run build`.
6. `.github/workflows/ci.yml`: jobs on push/PR to `main`:
   matrix `os: [ubuntu-latest, macos-latest, windows-latest]` ×
   `node: [20, 22]`; steps: checkout, setup-node (with npm cache), `npm ci`,
   `npm run lint` (ubuntu only), `npm run typecheck`, `npm run build`,
   `npm test`. Pin all actions to full commit SHAs (ADR-005 / issue 40 hardens further).
7. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `.skillet/`.
8. Do NOT modify or delete the existing `README.md` or anything under `docs/`.

## Acceptance Criteria

- [ ] `npm ci && npm run build && npm test && npm run typecheck && npm run lint` all pass locally
- [ ] `node dist/cli.js` prints `skillet 0.0.0` and exits 0
- [ ] CI workflow is green on all 6 matrix cells
- [ ] `npm pack --dry-run` lists only `dist/`, `schemas/` (if present), `package.json`, `README.md`, `LICENSE`
- [ ] No runtime dependency outside the ADR-003 list

## Validation

Run the commands in Acceptance Criteria; attach output. Verify `git status`
is clean after build (dist is ignored).

## Dependencies

None (first issue).

## Non-goals

Real command parsing (issue 17); release/publish workflow (issue 40); README
rewrite (issue 39).

## Design References

- DESIGN.md section 5 (module layout, dependency budget)
- ADR-003 (stack), ADR-005 (pinning), ADR-006 (package name + fallback)
