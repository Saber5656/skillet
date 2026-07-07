# Title

tests/helpers: sandbox environment, git fixture builder, index fixture server

## Summary

Build the three shared test utilities every subsequent issue's tests rely on:
a fully sandboxed skillet environment, a programmatic git repository builder,
and a local HTTP server serving aggregate-index fixtures with ETag semantics.

## Context

DESIGN.md 16: tests must not touch the real network or the user's home
directory. `SKILLET_HOME` (issue 03) provides the sandbox seam. Git fixtures
must be able to express hostile content (symlinks, weird modes) to power the
security suite (issue 37).

## Scope

- `tests/helpers/sandbox.ts`, `tests/helpers/gitfixture.ts`,
  `tests/helpers/indexserver.ts`
- `tests/helpers/helpers.test.ts` (self-tests)

## Detailed Requirements

1. `sandbox.ts`:
   ```ts
   export interface Sandbox { dir: string; home: string; skilletHome: string;
     projectDir: string; env: Record<string,string>;   // HOME, SKILLET_HOME, SKILLET_INDEX_URL?, GIT_* scrubs, NO_COLOR=1, SKILLET_YES?
     cleanup(): Promise<void>; }
   export function makeSandbox(t?: { indexUrl?: string; yes?: boolean }): Promise<Sandbox>;
   ```
   Uses `fs.mkdtemp(os.tmpdir())`; `env` also sets `GIT_CONFIG_NOSYSTEM=1`,
   `GIT_TERMINAL_PROMPT=0`, `HOME`, `XDG_CONFIG_HOME` under the sandbox, and
   `GIT_AUTHOR_*`/`GIT_COMMITTER_*` fixed identities for deterministic SHAs.
2. `gitfixture.ts`:
   ```ts
   export interface FixtureSpec {
     files: Record<string, string | { content: string; mode?: "100644" | "100755" }>;
     symlinks?: Record<string, string>;   // path -> target (created with git update-index or on-disk ln)
     tag?: string; branch?: string;       // defaults: branch "main"
   }
   export interface GitFixture { dir: string; url: string;   // file:// URL
     commits: string[]; addCommit(spec: FixtureSpec): Promise<string>;  // returns sha
     tagCommit(tag: string, sha: string): Promise<void>; }
   export function makeGitRepo(base: string, initial: FixtureSpec): Promise<GitFixture>;
   ```
   Implemented by spawning the system `git` (helper `run(cmd, args, cwd, env)`);
   symlink entries created via `ln -s` on POSIX and `git update-index
   --cacheinfo 120000` fallback so they exist on Windows too. Deterministic
   commits (fixed dates via `GIT_AUTHOR_DATE=2026-01-01T00:00:00Z`).
   Note: production code forbids `file:` transports (DESIGN 10.2); tests enable
   the documented test-only env flag `SKILLET_ALLOW_FILE_GIT=1` — add that flag
   to the gitfetch design contract here as a helper constant so issue 12
   implements it.
3. `indexserver.ts`:
   ```ts
   export interface IndexServer { url: string;   // http://127.0.0.1:<port>/index.json
     setIndex(doc: unknown): void;               // bumps ETag
     requests: { etagHits: number; full: number };
     close(): Promise<void>; }
   export function startIndexServer(initial: unknown): Promise<IndexServer>;
   ```
   `node:http`, binds 127.0.0.1:0; supports `If-None-Match` → 304; strong ETag
   = sha256 of body. (Client accepts `http://127.0.0.1`/`http://localhost` only
   when `SKILLET_INDEX_URL` env override is used — record this as a contract
   for issue 11.)
4. All helpers OS-portable; self-tests create a repo with two commits + a tag,
   assert stable SHAs across two runs in the same environment, and exercise the
   304 path.

## Acceptance Criteria

- [ ] Self-tests green on ubuntu/macos/windows CI
- [ ] Two sandboxes never share state; cleanup removes everything (checked)
- [ ] Fixture SHAs deterministic within a run environment (same spec → same sha)
- [ ] Index server: second fetch with ETag yields 304 and `etagHits === 1`

## Validation

`npm test -- helpers` green on the full CI matrix.

## Dependencies

01, 02, 03.

## Non-goals

Production code changes (except documenting the two env contracts above for
issues 11/12); e2e runner (issue 38).

## Design References

- DESIGN.md 16 (testing strategy), 10.2 (transport allowlist), 7.6 (ETag caching)
