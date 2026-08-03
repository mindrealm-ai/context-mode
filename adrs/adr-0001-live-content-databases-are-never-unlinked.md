---
id: adr-0001-live-content-databases-are-never-unlinked
title: "Live content databases are never unlinked"
status: accepted
version: 1
created: 2026-08-03
updated: 2026-08-03
supersedes: []
superseded_by: []
---

# Live content databases are never unlinked

## Context

The content store is intentionally shared by multiple context-mode processes. In 1.0.169,
`cleanupStaleContentDBs` promotes a WAL file older than one hour to `shouldClean = true` and
unlinks the database, WAL, and SHM files without proving that no process holds them open.
Linux permits the unlink while a process still owns the inode. A later process then creates a
new database at the same pathname, splitting one logical store across unrelated file identities.
The same runtime uses a synchronous busy-wait for SQLite retry delays, so persistent contention
can consume a full CPU core instead of yielding.

## Decision Drivers

- Preserve the supported multi-process content-store contract.
- Make destructive cleanup require ownership evidence, not an age heuristic.
- Bound CPU, wall time, and failure behavior during contention and IO errors.
- Keep recovery observable and reproducible under tests.
- Keep installation lifecycle-aware so one logical host registration starts one server.

## Considered Options

| Option | Advantages | Disadvantages |
|---|---|---|
| Keep age-based unlink cleanup and add CPU quotas | Small change; caps total CPU | Leaves live-inode corruption possible and masks the defect |
| Return to a single-writer global lock | Simplifies ownership | Breaks legitimate multi-window use and reverses the shared-store contract |
| Never unlink a possibly live database; use explicit ownership plus bounded yielding recovery | Preserves concurrency, prevents pathname/inode split, and makes failure bounded | Requires ownership records, regression tests, and lifecycle reconciliation |
| Replace SQLite | Avoids these exact file semantics | Large redesign unrelated to the immediate defects |

## Decision

context-mode never unlinks a content database, WAL, or SHM file unless it has established that
no live process owns the database and has acquired the cleanup exclusion required by the
implementation. Age is only a candidate-selection hint. Unknown ownership prevents deletion.
SQLite contention waits yield the CPU and are bounded; deleted-file, replaced-file, and IO-error
states recover once through an explicit reopen path or fail with a terminal diagnostic. The
supported Codex installation shape is exactly one lifecycle-aware plugin registration; the
installer diagnoses and reconciles duplicate explicit MCP registrations.

## Consequences

- Positive: a cleanup pass cannot split live processes across different databases at one path.
- Positive: retry behavior no longer converts SQLite contention into sustained core saturation.
- Positive: operators can identify the owning component and configuration defect before
  stopping a process.
- Negative: stale files may remain when ownership cannot be proved; safety takes precedence
  over eager disk reclamation.
- Negative: lifecycle and multi-process stress coverage becomes a release requirement.
- Neutral: a CPU quota remains a useful containment layer, but never evidence that the runtime
  defect is fixed.

## Confirmation

The governing Spec's live-owner, retry-CPU, IO-recovery, duplicate-registration, and sustained
stress verification obligations all pass on the release artifact. The old implementation must
fail at least the live-unlink and retry-CPU regression tests.
