# Title

skillet verify: integrity audit of materialized skills against the lockfile

## Summary

Implement `skillet verify`: rehash every receipt-tracked install and compare
against the lockfile, reporting OK/MODIFIED/MISSING/UNTRACKED per (package,
agent), with exit code 6 on any drift — the CI tamper/drift gate.

## Context

DESIGN 14 (exit 6 = drift) and 12.2 (verify detects divergence between lock
and materialized content). Verify is read-only: it never repairs (repair =
`skillet install` re-materializing, guided by the report).

## Scope

- `src/cli/commands/verify.ts`
- `tests/cli/verify.test.ts`

## Detailed Requirements

1. Synopsis: `skillet verify [--scope|-g] [--agents ids…]` (agents filter
   restricts which receipts are checked; default all receipts in scope).
2. Per lock entry × receipt: recompute `hashLocalDir` (issue 14) on the
   receipt dir and classify:
   - `ok`: dir exists, hash == lock integrity (also == receipt integrity)
   - `modified`: dir exists, hash != lock integrity (report both hashes,
     and whether it matches the receipt instead — distinguishes "edited after
     install" from "installed version no longer in lock")
   - `missing`: receipt exists, dir absent
   - `untracked`: receipt exists but package absent from lock (stale state)
   Lock entries with no receipt for a configured agent → `not-installed`
   (informational, does not affect exit code — install owns that gap; list
   shows it too).
3. Windows note: hashes were computed from git modes at install; local rehash
   on Windows yields all-`f` classes. To keep verify sound cross-OS, receipts
   additionally store `integrityNoExec` — REJECTED (schema stays DESIGN 7.3).
   Instead: on Windows, verify recomputes with classes taken from the LOCK's
   recorded source when available… ALSO rejected (complexity). Normative v1
   rule: on Windows, when a hash mismatch is solely explainable by exec-class
   bits, verify reports `ok-execbits` (counts as ok, printed dimmed). Implement
   by recomputing the manifest string with all classes forced to `f` for both
   sides and comparing; document this in code and in the command help.
4. Human output: table `PACKAGE  AGENT  STATUS  DETAIL`; summary line
   `N ok, M modified, K missing, J untracked`; exit 6 if M+K+J > 0 else 0.
   `--json`: `{ results: [{ id, agent, dir, status, expected, actual }],
   summary }`.
5. Performance: hash sequentially (v1); a `verbose` line per package when
   `--verbose`.

## Acceptance Criteria

- [ ] Clean install → all ok, exit 0
- [ ] Edit one file in one agent copy → that (package, agent) `modified`, others ok, exit 6
- [ ] Delete a copy → `missing`, exit 6
- [ ] Receipt for a package no longer in lock → `untracked`, exit 6
- [ ] Lock entry with no receipt → `not-installed`, exit unaffected
- [ ] exec-bit-only difference on Windows fixture → `ok-execbits`, exit 0 (POSIX: chmod change → `modified`)

## Validation

`npm test -- verify` green on CI matrix.

## Dependencies

10, 14, 16, 17.

## Non-goals

Auto-repair (`install` re-materializes); verifying unmanaged dirs; verifying
the lockfile against the index (that is publish/CI territory, issue 33).

## Design References

- DESIGN.md 14 (exit codes), 12.2 (lockfile trust), section 8 (hash), ADR-007
