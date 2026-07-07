# Title

indextools/build.ts: aggregate index.json builder with --check mode

## Summary

Implement the deterministic builder that merges `registry/**/*.json` into the
aggregate `index.json` clients fetch, plus a `--check` mode verifying the
committed aggregate matches the registry (PR gate).

## Context

DESIGN 7.6 defines the aggregate document; ADR-002 the publication flow
(CI builds and commits on merge to main). Byte-determinism matters: the
committed aggregate must be reproducible from the registry alone or `--check`
gates fail spuriously.

## Scope

- `src/indextools/build.ts`
- `tests/indextools/build.test.ts`

## Detailed Requirements

1. Export:
   ```ts
   export function buildAggregate(repoDir: string, opts: { generatedAt: string | null }): AggregateIndex;
   // generatedAt injected (CI passes commit timestamp of HEAD: `git log -1 --format=%cI`); null → omit? NO: field required by schema — null → "1970-01-01T00:00:00Z" sentinel forbidden; instead:
   // normative rule: generatedAt = the HEAD commit timestamp, so rebuilding the same commit yields identical bytes.
   export function writeAggregate(repoDir: string, agg: AggregateIndex): void;  // <repoDir>/index.json via stable jsonio
   export function checkAggregate(repoDir: string, agg: AggregateIndex): { upToDate: boolean; diffSummary: string | null };
   ```
2. Merge semantics: read every `registry/<owner>/<skill>.json` (already
   validated by issue 33 in CI order — builder still schema-parses
   defensively), sort packages by `name` asc, sort each `versions` array by
   semver desc; the aggregate carries entries verbatim otherwise.
3. Determinism (normative): same registry tree + same HEAD timestamp →
   byte-identical `index.json` (stable serialization from issue 09's jsonio;
   `generatedAt` from the injected commit timestamp, never wall clock).
4. `checkAggregate` compares canonical bytes of the freshly built document
   (with `generatedAt` taken from the EXISTING committed aggregate, so check
   mode ignores timestamp differences) against the committed file; returns a
   unified-diff-style first-20-lines summary on mismatch.
5. Size guard: refuse to build an aggregate > 50 MiB (matches client cap,
   issue 11) with rule `aggregate-too-large` error.

## Acceptance Criteria

- [ ] Fixture registry (3 entries) builds the expected aggregate (snapshot); build twice → identical bytes
- [ ] Version ordering: shuffled input versions serialize semver-desc
- [ ] `--check` semantics: committed stale aggregate detected; timestamp-only difference does NOT fail check
- [ ] Malformed entry file → E_SPEC_INVALID naming the file
- [ ] generatedAt equals the injected timestamp verbatim

## Validation

`npm test -- indextools-build` green.

## Dependencies

05, 09 (jsonio), 33 (shared fixtures; can proceed in parallel once fixtures exist).

## Non-goals

Sharding/pagination (v2); publishing/committing (index repo CI's job, issue 36).

## Design References

- DESIGN.md 7.6 (aggregate — normative), section 11 (CI flow), ADR-002
