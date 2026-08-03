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
- Make every destructive filesystem transition use the same ownership proof.
- Keep cleanup observable and reproducible under multi-process races.

## Considered Options

| Option | Advantages | Disadvantages |
|---|---|---|
| Keep age-based unlink cleanup and add CPU quotas | Small change; caps total CPU | Leaves live-inode corruption possible and masks the defect |
| Return to a single-writer global lock | Simplifies ownership | Breaks legitimate multi-window use and reverses the shared-store contract |
| Never unlink or rename a possibly live database; require explicit ownership and exclusion | Preserves concurrency and prevents pathname/inode split across every destructive path | Requires ownership records and multi-process regression tests |
| Replace SQLite | Avoids these exact file semantics | Large redesign unrelated to the immediate defects |

## Decision

context-mode never deletes, unlinks, renames, quarantines, or replaces a content database, WAL,
or SHM file unless it has proved that no live process owns the database and holds cross-process
exclusion through the entire transition. Age is only a candidate-selection hint. Closing the
current process's handle does not prove that sibling owners are absent. Unknown ownership
prevents the transition.

## Consequences

- Positive: a cleanup pass cannot split live processes across different databases at one path.
- Negative: stale files may remain when ownership cannot be proved; safety takes precedence
  over eager disk reclamation.
- Negative: corruption quarantine and recovery can fail terminally when sibling ownership is
  unknown; preserving one database identity takes precedence over automatic recovery.

## Confirmation

The governing Spec's live-owner and destructive-transition race obligations pass on the release
artifact. The old implementation fails the stale-WAL live-unlink control, while the patch proves
that unlink, rename, quarantine, and replacement all stop on live or unknown ownership.
