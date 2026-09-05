# Review resolution record

- Repository: `Saber5656/skillet`
- Pull request: #41
- Parent head observed before this addendum: `dabdc57e7bee44f420beb3571e2185fa8f893aa2`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNj9aM6O8VjL`

### Preserve agent selection source for target planning

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8VjL` identifies this contract gap.
- Normative resolution: Return adapter selections with provenance (`detected` or `explicit`) so `planTargets` can avoid creating user runtime base directories for adapters that were not requested and have no detected target.
- Focused verification before resolving this thread: Plan detected-only, explicit-only, and mixed adapter inputs and assert target creation follows the provenance rules.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8VjU`

### Keep raw git URL strings out of manifests

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8VjU` identifies this contract gap.
- Normative resolution: Make manifest parsing accept only the documented structured git spec; raw URL forms with fragment/path syntax are rejected or normalized before persistence, so unsupported syntax cannot enter `skillet.json`.
- Focused verification before resolving this thread: Parse raw URL, structured, and invalid specs and assert only the documented form is persisted.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8VjZ`

### Declare pending identities in the plan type

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8VjZ` identifies this contract gap.
- Normative resolution: Add `pendingIdentity: true` (or an equivalent discriminated plan state) to the exported PlannedPackage contract, with the finalize-after-stage transition defined.
- Focused verification before resolving this thread: Typecheck a direct-git add plan and assert pending identity is represented until staging resolves the package id.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vjd`

### Preserve explicit branch refs for updates

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vjd` identifies this contract gap.
- Normative resolution: Persist whether a direct-git dependency was requested by branch/ref versus immutable SHA; branch entries re-resolve on update while SHA entries remain pinned.
- Focused verification before resolving this thread: Add branch, tag, and SHA specs, update them, and assert only the named mutable ref changes.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vjh`

### Store enough data for Windows verify

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vjh` identifies this contract gap.
- Normative resolution: Store a canonical per-file manifest or equivalent file-class map in the lock/receipt so Windows verification can recompute the `f`-normalized integrity rather than relying only on an aggregate string.
- Focused verification before resolving this thread: Install a tree containing executable-bit variants on Windows-compatible fixtures and assert verify recomputes the expected hash.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vjr`

### Define the drift-recovery force flag

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vjr` identifies this contract gap.
- Normative resolution: Add `add --force` to the CLI contract, require an explicit confirmation/receipt replacement path, and limit it to the documented receipt-hash drift recovery.
- Focused verification before resolving this thread: Run add against a drifted managed tree with and without `--force` and assert the default refuses while force follows the guarded recovery path.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vjx`

### Forbid normal removal of published versions

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vjx` identifies this contract gap.
- Normative resolution: Make released versions immutable even when deprecated; deprecation changes metadata, while removal requires a separately authorized migration/administrative path outside the normal index PR.
- Focused verification before resolving this thread: Attempt normal removal of deprecated and unreleased versions and assert only the permitted case succeeds.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vj4`

### Allow repository-root skill paths

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vj4` identifies this contract gap.
- Normative resolution: Treat an empty relative skill path as the repository root, while retaining path containment and rejecting absolute/traversal paths.
- Focused verification before resolving this thread: Resolve root, nested, absolute, and `..` paths and assert root is accepted only for the exact empty relative representation.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8Vj8`

### Make the CI workflow reusable before calling it

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8Vj8` identifies this contract gap.
- Normative resolution: Declare `workflow_call` and its explicit inputs/secrets on `ci.yml` before release workflow reuse; preserve push/pull request triggers as separate entry points.
- Focused verification before resolving this thread: Parse the workflow graph and assert the release `uses:` target has a valid workflow_call interface.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8VkH`

### Preserve the Git system-config scrub

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8VkH` identifies this contract gap.
- Normative resolution: Set `GIT_CONFIG_NOSYSTEM=1` for every Git invocation in the adapter/materializer path, in addition to the existing prompt and command flags.
- Focused verification before resolving this thread: Install a hostile system Git config fixture and assert URL rewrites and system settings cannot affect resolution.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj9aM6O8VkM`

### Report unsafe installed trees as verify drift

- Finding: The existing review thread `PRRT_kwDOTNj9aM6O8VkM` identifies this contract gap.
- Normative resolution: Map `E_UNSAFE_TREE` and any symlink/non-regular-file violation to a deterministic verify-drift result with remediation, rather than letting it escape as an unclassified exception.
- Focused verification before resolving this thread: Add a symlink and special-file fixture to an installed tree and assert verify reports drift and a safe remediation.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.