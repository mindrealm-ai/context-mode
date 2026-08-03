---
id: adr-0004-database-opens-and-destructive-transitions-share-one-exclusion-protocol
title: "Database opens and destructive transitions share one exclusion protocol"
status: accepted
version: 1
created: 2026-08-03
updated: 2026-08-03
supersedes:
  - adr-0001-live-content-databases-are-never-unlinked
superseded_by: []
---

# Database opens and destructive transitions share one exclusion protocol

## Context

ADR 0001 required no-live-owner proof and cross-process exclusion for destructive database
transitions. That ordering still permitted a time-of-check/time-of-use race if an ordinary opener
could create a new live owner after proof but before unlink, rename, quarantine, or replacement.
Safety requires openers and destructive transitions to participate in opposite sides of one
protocol, not two independent checks.

## Decision Drivers

- Preserve legitimate concurrent readers and writers.
- Make a no-live-owner proof remain true through the destructive transition.
- Fail closed when ownership or exclusion is unavailable.
- Prove the ordering under a deterministic race.

## Considered Options

| Option | Advantages | Disadvantages |
|---|---|---|
| Check owners, then acquire a cleanup lock | Small cleanup change | A new opener can enter between the check and lock |
| Lock only destructive transitions | Serializes cleanup workers | Ordinary opens can still race the transition |
| Shared ownership for opens and exclusive exclusion for destructive transitions | Preserves normal concurrency and closes the race | Every open and close path must participate correctly |

## Decision

Every ordinary content-database open acquires the compatible shared side of the ownership/exclusion
protocol before opening the pathname and retains it until the database handle closes. Every delete,
unlink, rename, quarantine, replacement, or fresh-create recovery first acquires the exclusive side,
then proves no live owner and revalidates the pathname and file identities while still holding that
exclusive exclusion. It retains exclusion through the complete transition. Failure to acquire,
prove, or revalidate makes the transition terminal and leaves the files untouched.

## Consequences

- Positive: a new opener cannot enter between ownership proof and a destructive transition.
- Positive: the multi-process open contract remains available through compatible shared ownership.
- Negative: every open, close, cleanup, and recovery path must use the shared protocol.
- Neutral: stale files remain when the exclusive proof cannot be completed safely.

## Confirmation

The governing Spec runs a barrier-controlled race that pauses a destructive worker after candidate
selection, starts an ordinary opener, and proves either the opener owns the shared side first and the
transition stops or the transition owns the exclusive side first and the opener waits until the
completed identity is safe. Identity is revalidated inside exclusive exclusion for every destructive
transition class.
