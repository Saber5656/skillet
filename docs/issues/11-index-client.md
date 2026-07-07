# Title

core/indexclient.ts: aggregate index fetch, ETag cache, offline mode, queries

## Summary

Implement fetching the aggregate `index.json` over HTTPS with ETag
revalidation and local caching, plus the query functions (find, search
ranking) used by search/info/add/update.

## Context

DESIGN.md 7.6: clients GET the aggregate index, cache body+ETag under
`<cacheDir>/index/`, revalidate with `If-None-Match`, operate offline from
cache with a staleness warning (> 7 days). DESIGN 12.2/12.3: HTTPS only; the
URL comes exclusively from env/user config (never project files).

## Scope

- `src/core/indexclient.ts`
- `tests/core/indexclient.test.ts` (uses helpers from issue 08)

## Detailed Requirements

1. Export:
   ```ts
   export interface IndexFetchResult { index: AggregateIndex; fromCache: boolean;
     fetchedAt: Date; stale: boolean; }
   export function fetchIndex(env: Environment, opts: { refresh?: boolean; offlineOk?: boolean }): Promise<IndexFetchResult>;
   export function findPackage(idx: AggregateIndex, id: string): IndexEntry | null;   // exact, case-sensitive (ids are lowercase by schema)
   export function nearestNames(idx: AggregateIndex, id: string, max: number): string[]; // Levenshtein ≤ 2 on full id, then on skill segment
   export interface SearchHit { entry: IndexEntry; score: number; }
   export function searchIndex(idx: AggregateIndex, query: string, keywords: string[]): SearchHit[];
   ```
2. URL policy: allow `https://…` always; allow `http://127.0.0.1…` /
   `http://localhost…` ONLY when the URL came from the `SKILLET_INDEX_URL` env
   override (test seam per issue 08); anything else → `E_CONFIG`.
3. Cache files: `<cacheDir>/index/index.json`, `<cacheDir>/index/meta.json`
   (`{ etag, url, fetchedAt }`). If cached `url` differs from current
   `env.indexUrl`, ignore the cache (different registries must not bleed).
4. Fetch with global `fetch`, 30 s timeout via `AbortSignal.timeout`,
   `If-None-Match` when meta exists; 304 → cache body; 200 → validate with
   `AggregateIndexSchema` BEFORE writing cache (invalid → `E_SPEC_INVALID`,
   cache untouched); other statuses → `E_FETCH` with status in message.
5. Network failure handling: if `offlineOk` and cache exists → return cache
   with `stale` computed from `fetchedAt` (> 7 days) and `fromCache: true`;
   else `E_FETCH` with hint about connectivity/`skillet cache dir`.
   `refresh: true` forces revalidation (used by `update`); default behavior for
   read commands: use cache if `fetchedAt` < 1 h old without any request.
6. Search ranking (deterministic, documented): score = 100 exact name match;
   80 skill-segment exact; 60 name prefix; 40 substring in name; 20 substring
   in description; +10 per matched `--keyword` (must ALL match to be a hit);
   ties broken by name asc. Query matching is case-insensitive; deprecated
   entries rank last (score − 1000) and are flagged.
7. Body size cap 50 MiB (`E_SPEC_INVALID` beyond — DoS guard).

## Acceptance Criteria

- [ ] First fetch 200 → cached; second within 1 h → zero HTTP requests; after `refresh` → conditional request; server 304 path exercised (etagHits assertion)
- [ ] URL policy: `http://example.com` rejected E_CONFIG; `http://127.0.0.1:<port>` accepted only via env override
- [ ] Offline with warm cache returns data (`fromCache`, `stale` correct); offline cold → E_FETCH
- [ ] Registry switch (different URL, same cacheDir) ignores prior cache
- [ ] Ranking table test: fixture with 6 packages asserts exact ordering; deprecated ranks last
- [ ] Invalid index body never overwrites a valid cache

## Validation

`npm test -- indexclient` green; no real-network test anywhere (assert via
sandbox env `SKILLET_INDEX_URL` pointing at the fixture server).

## Dependencies

01, 02, 03, 05, 08.

## Non-goals

Version range selection (issue 19); search CLI rendering (issue 25).

## Design References

- DESIGN.md 7.6 (aggregate + caching), 12.2/12.3 (URL trust), 14 (search command)
