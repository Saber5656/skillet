# Title

core/gitfetch.ts (part 1) + core/spawn.ts: git availability, mirrors, commit fetch, ref resolution

## Summary

Implement the safe child-process wrapper and the git mirror layer: detect git,
create/reuse bare mirrors per source URL, ensure a commit exists locally
(with the fallback chain), and resolve remote refs to SHAs.

## Context

ADR-008 and DESIGN.md 10.2 fix the mechanism: bare mirror per URL under
`<cacheDir>/repos/<sha256(url)>/`; `git fetch origin <sha>` with fallbacks;
argument-vector spawns with scrubbed env, protocol allowlist, timeouts, and
validated refs (injection defenses, DESIGN 12.2).

## Scope

- `src/core/spawn.ts`, `src/core/gitfetch.ts` (mirror/fetch/ref parts)
- `tests/core/spawn.test.ts`, `tests/core/gitfetch.test.ts`

## Detailed Requirements

1. `spawn.ts`:
   ```ts
   export interface RunResult { stdout: Buffer; stderr: string; code: number; }
   export function run(cmd: string, args: string[], opts: { cwd?: string; env?: Record<string,string>;
     timeoutMs?: number; maxBuffer?: number; stdin?: Buffer }): Promise<RunResult>;  // rejects with E_INTERNAL wrapping on spawn errors; never invokes a shell
   ```
   Default timeout 60 000 ms (kills process group), maxBuffer 64 MiB.
2. `gitfetch.ts` exports:
   ```ts
   export function ensureGitAvailable(): Promise<{ version: string }>;   // E_GIT_MISSING if absent or < 2.30, message includes install hint
   export function mirrorDirFor(env: Environment, url: string): string;  // <cacheDir>/repos/<sha256hex(url)>
   export function ensureMirror(env: Environment, url: string): Promise<string>; // init --bare + remote add origin (idempotent); validates transport
   export function ensureCommit(env: Environment, url: string, sha: string): Promise<void>; // fallback chain below; E_FETCH on failure
   export function resolveRemoteRef(env: Environment, url: string, ref: string | null): Promise<string>; // null → HEAD; returns 40-hex; E_RESOLVE if not found
   ```
3. Transport allowlist in `ensureMirror`/`resolveRemoteRef`: `https://`,
   `ssh://`, scp-style `git@host:…`; `file://`/local paths allowed ONLY when
   `SKILLET_ALLOW_FILE_GIT=1` (test seam, issue 08). Everything else →
   `E_CONFIG`.
4. Git env scrubbing for every invocation: inherit user env for credentials
   BUT force `GIT_TERMINAL_PROMPT=0`; add config args
   `-c protocol.ext.allow=never -c protocol.file.allow=<user|never per flag>
   -c core.askPass=` ; refs validated with the issue-07 regex before use; SHAs
   validated `/^[0-9a-f]{40}$/`; `--` separates pathspecs everywhere applicable.
5. `ensureCommit` algorithm (stop at first success):
   1. `git cat-file -e <sha>^{commit}` in mirror → done (cache hit)
   2. `git fetch origin <sha>` (depth untouched)
   3. `git fetch origin +refs/heads/*:refs/heads/* +refs/tags/*:refs/tags/*`
      then re-check cat-file
   4. fail `E_FETCH` (message includes url, sha, last stderr; hint about host
      SHA-fetch support per DESIGN 15 known unknown)
   Retries: steps 2-3 retried once on network-class failures (exit code 128 +
   stderr matching /unable to access|could not resolve|timed out/i) after 2 s.
6. `resolveRemoteRef`: `git ls-remote origin <ref>` — precedence exact match on
   `refs/tags/<ref>` > `refs/heads/<ref>` > `HEAD` (for null); peeled tag
   (`^{}`) preferred over tag object SHA. 40-hex input returns itself without
   network.
7. Concurrency safety: per-mirror lockfile `<mirror>/skillet.lock-pid` via
   `mkdir` (EEXIST → wait/retry up to 30 s, then E_CACHE with hint about a
   crashed process and `skillet cache clean`).

## Acceptance Criteria

- [ ] Fixture repo (issue 08): resolveRemoteRef for branch, tag (annotated + lightweight), HEAD, and raw SHA all return the expected 40-hex
- [ ] ensureCommit succeeds for a SHA present only via fallback step 3 (simulate by fetching a repo whose server-side `uploadpack.allowAnySHA1InWant` is off — with file transport this is emulated by asserting step order via injected runner)
- [ ] Refusals: `ext::sh -c` URL, ref `--upload-pack=x`, ref with `..` → E_CONFIG/E_USAGE before any spawn (assert runner not called)
- [ ] git < 2.30 (mock `git --version` output) → E_GIT_MISSING
- [ ] Two concurrent ensureCommit calls on one mirror serialize (lock test)

## Validation

`npm test -- spawn gitfetch` green on CI matrix (Windows included — spawn uses
`windowsHide`, no shell).

## Dependencies

01, 02, 03, 08.

## Non-goals

Tree listing/extraction (issue 13); staging (issue 20); cache clean command
(issue 32).

## Design References

- DESIGN.md 10.2 (fetch strategy), 12.2 (injection defenses), ADR-008
