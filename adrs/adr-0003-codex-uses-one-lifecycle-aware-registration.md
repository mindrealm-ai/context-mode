---
id: adr-0003-codex-uses-one-lifecycle-aware-registration
title: "Codex uses one lifecycle-aware context-mode registration"
status: accepted
version: 1
created: 2026-08-03
updated: 2026-08-03
supersedes: []
superseded_by: []
---

# Codex uses one lifecycle-aware context-mode registration

## Context

Codex can load context-mode from the plugin and from an explicit MCP command. The explicit bare
command bypasses the plugin launcher, so one logical component can start twice with different
lifecycle behavior.

## Decision Drivers

- One logical registration starts one server lifecycle.
- Reconciliation must be inspectable, idempotent, reversible, and safe on ambiguous custom commands.
- Repository tests must not mutate a live operator configuration.

## Considered Options

| Option | Advantages | Disadvantages |
|---|---|---|
| Keep both registrations and rely on resource caps | Preserves existing files | Duplicate processes and lifecycle ambiguity remain |
| Keep only the explicit bare MCP command | Simple config | Bypasses supported plugin lifecycle |
| Converge to one lifecycle-aware plugin registration | One owner and launcher; plugin upgrades remain coherent | Requires reconciliation and live operator rollout |

## Decision

The supported Codex shape is exactly one context-mode plugin registration using its lifecycle-aware
launcher. The reconciler removes a recognized duplicate explicit entry after backup, leaves ambiguous
custom commands unchanged, supports dry-run and restore, and proves idempotency in an isolated config home.

## Consequences

- Positive: one logical context-mode registration has one lifecycle owner.
- Positive: live rollout can be backed up, verified, and rolled back.
- Negative: custom command variants require operator disposition rather than automatic mutation.
- Neutral: other agent hosts retain their own supported lifecycle adapters.

## Confirmation

Isolated reconciliation fixtures and the claim-gated live rollout both report one registration,
one process tree, lifecycle readiness, and successful backup/restore proof.
