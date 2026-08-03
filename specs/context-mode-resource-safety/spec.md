---
id: context-mode-resource-safety
prd: context-mode-resource-safety
prd_version: a85c0fd3e90c75e9ab5f510e5226b6624850265c
status: approved
version: 1
created: 2026-08-03
updated: 2026-08-03
dependencies: []
rollout_constraints:
  - "Do not recommend or install the fork until every verification obligation passes on the packaged artifact."
  - "Live user configuration is operator-owned; repository Tasks prove reconciliation against isolated config homes."
open_questions: []
---

# Spec — Context-mode resource safety

## Scope

Repair the context-mode SQLite cleanup and retry defects that can leave a Node process pinned at
100% CPU, add diagnosis and lifecycle reconciliation, and prove the packaged fork under
multi-process stress. The capability preserves context-mode's compression and retrieval value;
it does not replace or disable it.

## Current Behavior

In 1.0.169, `src/store.ts:cleanupStaleContentDBs` marks a content database for cleanup when its
WAL is non-empty and more than one hour old, without proving that the database has no live
owners. It then unlinks the main database, WAL, and SHM files. On Linux, a live process can keep
the deleted inodes open while a new process creates different files at the same pathname.
`src/db-base.ts:withRetry` waits for SQLite contention using a synchronous `Date.now()` loop,
which consumes a core for the duration of every retry. The Codex plugin and a separate explicit
MCP entry can also start two servers, and the bare command bypasses the plugin launcher.

## Required Behavior

- **CMRS-REQ-1 — Cleanup ownership.** Content-database cleanup treats age as a candidate hint,
  never as proof of abandonment. It unlinks files only after proving no live owner exists and
  acquiring an exclusion that prevents a new owner from opening the candidate during cleanup.
  Unknown ownership leaves the files untouched and emits a diagnostic.
- **CMRS-REQ-2 — Database identity.** Every open content database records enough identity to
  detect that its pathname has been replaced or its inode has been unlinked. A detected identity
  change prevents further normal operations and enters the bounded recovery path.
- **CMRS-REQ-3 — Yielding contention.** SQLite contention waits use SQLite's bounded timeout or
  a yielding delay. No retry path polls the clock or spins synchronously.
- **CMRS-REQ-4 — Bounded IO recovery.** `SQLITE_IOERR`, deleted-file, replaced-file, corruption,
  and lock exhaustion have explicit classifications. Recoverable cases close and reopen at most
  once under exclusion; unrecoverable cases fail with the database path, classification, and
  next safe operator action. No class retries indefinitely.
- **CMRS-REQ-5 — Diagnosis.** A read-only doctor reports context-mode PIDs and ancestry, package
  version and launch path, database paths and identities, deleted open files, registration
  sources, duplicate logical registrations, lifecycle readiness, CPU time/utilization, RSS, and
  unavailable observations. It distinguishes context-mode from Headroom.
- **CMRS-REQ-6 — Registration reconciliation.** The Codex installer/reconciler detects plugin,
  explicit MCP, and bare-launcher registrations. Dry-run reports the target shape. Apply mode
  converges an isolated config home to exactly one lifecycle-aware plugin registration and is
  idempotent and reversible.
- **CMRS-REQ-7 — Release stress.** The release gate runs concurrent readers/writers, cleanup,
  repeated starts/stops, abandoned-parent simulation, forced SQLite locks, and idle observation
  against the packaged artifact. It records process, CPU, memory, database-identity, and error
  evidence.

## Invariants

- A database with a live or unknown owner is never unlinked.
- One pathname never represents two live content-database identities by action of context-mode.
- Contention consumes bounded wall time and does not intentionally consume CPU while waiting.
- Cleanup, recovery, and reconciliation are fail-closed and leave evidence.
- A quota, lower priority, or restart is containment, never a passing repair signal.
- Missing telemetry is `unavailable`, never zero.

## Interfaces

- `src/store.ts` — stale content-store cleanup and content-store lifecycle.
- `src/db-base.ts` — SQLite open, identity, retry, close, and recovery behavior.
- `src/server.ts` and `src/lifecycle.ts` — server startup, shutdown, and lifecycle diagnostics.
- `start.mjs`, Codex adapter/install surfaces, and plugin manifest — canonical launcher and
  registration reconciliation.
- Doctor output — stable human-readable and JSON shapes for read-only diagnosis.

## Failure Behavior

- Ownership cannot be established: skip deletion, record `ownership=unknown`, continue startup.
- Exclusion cannot be acquired within its bound: skip deletion and record contention.
- Path identity changes while open: stop normal database operations, close the handle, attempt
  the single classified recovery, then either reopen safely or fail terminally.
- Reconciliation finds an ambiguous user-managed command: report it and make no change.
- Stress evidence is incomplete: release gate fails; absence is not a passing measurement.

## Migration Behavior

The fork retains existing on-disk schemas. The installer first supports dry-run, backs up an
isolated target configuration before changing it, and can restore that backup. Existing content
databases are opened in place; no eager migration or deletion is permitted.

## Acceptance Criteria

- **CMRS-AC-001.** WHEN process A holds a content database open and process B runs stale cleanup
  after the age threshold, the cleanup SHALL keep the main, WAL, and SHM identities linked and
  SHALL report a live owner.
- **CMRS-AC-002.** WHEN cleanup cannot prove whether a process owns a candidate database, it
  SHALL leave every candidate file untouched and report unknown ownership.
- **CMRS-AC-003.** WHEN SQLite remains locked through every retry, context-mode SHALL terminate
  the operation within the configured bound and retry CPU time SHALL remain below the gate's
  explicit threshold.
- **CMRS-AC-004.** WHEN an open database path is replaced, unlinked, or produces a classified
  IO error, context-mode SHALL recover once or fail terminally without an unbounded retry.
- **CMRS-AC-005.** WHEN doctor runs against a hot context-mode process with deleted SQLite
  files, it SHALL identify the process as context-mode, list the deleted files and registration
  source, and SHALL perform no mutation.
- **CMRS-AC-006.** WHEN an isolated Codex config contains both plugin and explicit MCP
  registrations, reconciliation SHALL converge it to one lifecycle-aware plugin registration;
  a second run SHALL produce no diff.
- **CMRS-AC-007.** WHEN the packaged artifact completes the release stress scenario, it SHALL
  record zero live-file unlink events, zero unbounded retry events, zero orphan servers, and idle
  CPU below the gate's declared threshold.
- **CMRS-AC-008.** WHEN any observation cannot be collected, evidence SHALL mark it unavailable
  and the applicable release assertion SHALL fail rather than treating it as zero.

## Verification Requirements

- **CMRS-VO-001.** Add a two-process Linux regression that reproduces the old stale-WAL unlink
  behavior. Pin the old implementation as a failing control and prove CMRS-AC-001..002 on the
  patch using file identity plus open-handle inspection.
- **CMRS-VO-002.** Force SQLite lock contention and measure process CPU time and wall time around
  retry. Assert no source-level clock-spin remains and prove CMRS-AC-003.
- **CMRS-VO-003.** Inject replaced-path, deleted-open-file, `SQLITE_IOERR`, corruption, and lock
  exhaustion cases; prove the one-recovery-or-terminal matrix in CMRS-AC-004.
- **CMRS-VO-004.** Run doctor against fixtures and live test processes representing context-mode,
  Headroom, duplicate registration, bare launch, deleted files, and unavailable metrics; snapshot
  both text and JSON output and prove CMRS-AC-005 and CMRS-AC-008.
- **CMRS-VO-005.** Reconcile isolated Codex homes for plugin-only, explicit-only,
  plugin-plus-explicit, ambiguous custom command, and repeat-apply cases; prove CMRS-AC-006 and
  backup/restore behavior.
- **CMRS-VO-006.** Build the release artifact, run the sustained lifecycle and multi-process
  stress suite against that artifact, and preserve the evidence bundle proving CMRS-AC-007.

## Dependencies

- `adrs/adr-0001-live-content-databases-are-never-unlinked.md`.
- Upstream behavior in context-mode 1.0.169, including the shared multi-writer contract.

## Rollout Constraints

As declared in front matter. The final Task packages and proves the fork; applying it to the
founder's live Codex installation is a separate operator action after that proof.

## Open Questions

None.
