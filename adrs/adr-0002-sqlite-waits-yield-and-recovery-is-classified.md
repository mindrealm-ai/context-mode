---
id: adr-0002-sqlite-waits-yield-and-recovery-is-classified
title: "SQLite waits yield and recovery is classified"
status: accepted
version: 1
created: 2026-08-03
updated: 2026-08-03
supersedes: []
superseded_by: []
---

# SQLite waits yield and recovery is classified

## Context

`withRetry` implements SQLite backoff with synchronous clock polling. A locked database therefore
spends CPU while it waits. Deleted, replaced, corrupt, and generic IO failures also need different
recovery behavior; one broad retry or rename path can repeat the failure or split live database identity.

## Decision Drivers

- Waiting must yield CPU and terminate within a measured bound.
- Each failure class must have one predictable outcome.
- Recovery must preserve the no-live-owner and exclusion decision in ADR 0001.
- Automatic recovery must never loop.

## Considered Options

| Option | Advantages | Disadvantages |
|---|---|---|
| Keep synchronous retry and cap CPU | Minimal code change | Burns the cap and hides the mechanism |
| Retry every SQLite/IO failure uniformly | Simple API | Repeats permanent failures and risks destructive recovery races |
| Use yielding bounded waits plus an explicit recovery matrix | Predictable CPU, wall time, and outcomes | More classified tests and terminal errors |

## Decision

Lock contention yields through SQLite timeout or asynchronous delay and terminates after at most
35 seconds with no reopen. Deleted or replaced path identity closes the local handle and may reopen
once only after ADR 0001 ownership proof and exclusion; a competing or unknown owner makes it
terminal. Corruption/NOTADB may quarantine and recreate once only under the same proof/exclusion.
Generic `SQLITE_IOERR` is terminal and performs no automatic destructive transition. No class
recovers more than once.

## Consequences

- Positive: contention no longer intentionally consumes a core while waiting.
- Positive: tests can assert one outcome for each failure class.
- Negative: generic IO failures require operator diagnosis rather than optimistic retry.
- Neutral: CPU quotas remain containment and are outside the correctness decision.

## Confirmation

Forced-lock evidence shows at most 35 seconds wall time and at most 50 milliseconds of process CPU
time attributable to a 3-second synthetic wait. Injection tests prove the matrix and one-recovery cap.

