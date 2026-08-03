---
id: context-mode-resource-safety
title: "Keep context compression useful without runaway CPU or live-database corruption"
status: approved
version: 2
created: 2026-08-03
updated: 2026-08-03
risk: high
specs:
  - specs/context-mode-resource-safety/spec.md
decisions_required:
  - adr-0004-database-opens-and-destructive-transitions-share-one-exclusion-protocol
  - adr-0002-sqlite-waits-yield-and-recovery-is-classified
  - adr-0003-codex-uses-one-lifecycle-aware-registration
---

# Keep context compression useful without runaway CPU or live-database corruption

## Problem

context-mode provides valuable output routing, sandboxed analysis, persistent retrieval, and
session accounting, but release 1.0.169 can consume a full CPU core indefinitely. Two source
paths can produce that failure. `cleanupStaleContentDBs` treats an old WAL modification time as
proof that a shared content database is unused and unlinks its database, WAL, and SHM files.
Another process can still hold those inodes open, so subsequent processes open a different
database at the same pathname while the first process continues operating on deleted files.
When SQLite reports contention, `withRetry` implements its backoff by spinning synchronously.
The result observed on 2026-08-03 was a `node-MainThread` context-mode process pinned at 100%
CPU with deleted SQLite files held open. CPU quota contained the drain but did not fix it.

The current Codex installation can also register context-mode twice: once as a plugin and once
as an explicit MCP command. The explicit command starts the bare CLI instead of the plugin's
lifecycle-aware launcher, so duplicate servers and orphaned lifecycles remain possible even
after the database defects are repaired.

## Desired Outcomes

- A live SQLite database, WAL, or SHM file is never unlinked by age-based cleanup.
- SQLite contention and recovery paths yield the CPU and terminate within a bounded time.
- IO errors caused by a replaced or unavailable database surface a bounded recovery or a clear
  terminal diagnostic instead of a hot loop.
- The installer/reconciler produces one lifecycle-aware context-mode registration per host and
  detects duplicate or bare-launcher registrations.
- A doctor command identifies PID ownership, open/deleted database files, duplicate
  registrations, lifecycle ancestry, and resource usage without changing the machine.
- Release and stress tests prove multi-process safety and sustained idle CPU behavior before a
  build is recommended for installation.

## Non-Goals

- Removing context-mode, Headroom, RTK, LSP integration, or token compression from the harness.
- Treating a CPU quota, lower process priority, or periodic restart as the permanent fix.
- Replacing SQLite or redesigning the retrieval feature set.
- Editing a user's live Codex configuration from a repository worktree.
- Claiming battery-energy savings from CPU measurements alone.

## Constraints

- Preserve multi-window and multi-process access to the shared content store.
- Every cleanup, delete, rename, quarantine, or replacement must establish no-live-owner proof and
  hold cross-process exclusion; file age and closing only the current handle are never ownership evidence.
- Retries use SQLite's bounded wait or an asynchronous/yielding delay, never a JavaScript
  busy-wait.
- Unknown ownership or telemetry is represented as unknown and fails cleanup closed.
- Installation changes are idempotent, reversible, and tested against an isolated home/config
  directory before an operator applies them to a live machine.
- Tests cover Linux semantics where unlinking an open file leaves a process on a deleted inode.

## Success Measures

- A deterministic two-process regression test keeps the first process's live database files
  linked while cleanup runs in the second process, and both processes retain one database
  identity.
- A forced-lock test records bounded wall time and negligible retry CPU time; the old
  implementation fails the CPU assertion.
- A recovery test for `SQLITE_IOERR`, deleted-file, and replaced-file conditions exits or
  reopens safely without an unbounded loop.
- A sustained stress run with concurrent readers/writers and repeated lifecycle restarts
  records zero database replacement events, zero orphan servers, and idle CPU returning below
  the specified test threshold.
- The configuration reconciler converges plugin-plus-explicit registration to exactly one
  lifecycle-aware registration and reports the change it would make in dry-run mode.

## Required Human Decisions

The founder approved creating `mindrealm-ai/context-mode`, patching the fork, and keeping
implementation in fresh Task sessions on 2026-08-03. Applying a proven artifact to live Codex
configuration remains a separate decision: a Reviewer Task submits the exact artifact, allowed
paths, proposed diff, and rollback to the human review queue, and the rollout Task cannot become
claimable until a founder-certified approval completes that prerequisite.
