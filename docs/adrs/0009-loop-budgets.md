# ADR-0009: Loop budgets for implementer runs

**Status:** Accepted
**Date:** 2026-04-22

## Context

An implementer that retries forever wastes tokens and may eventually
stumble into undesired behaviors by chance. We need hard limits, but
they should be generous enough that realistic tasks finish.

## Decision (v1 starting values — tune after data)

Budgets enforced by Claude Code hooks:

| Budget | Limit | Rationale |
|--------|-------|-----------|
| Wall-clock | 45 min | Includes CI wait. A typical task finishes in 5-15 min. |
| Tool calls (total) | 300 | Generous for multi-file edits. |
| File edits | 75 | Each edit is a distinct `Edit`/`Write`. |
| Validate invocations | 8 | Encourages thoughtful retries, not machine-gun retry. |
| Retries after validate=fail | 5 | Orthogonal safeguard; an early "give up" path. |
| Input tokens | 750k | ~half the 1M context for headroom. |
| Output tokens | 200k | Generous for plan+patch. |
| CI poll timeout | 10 min | Per-run override supported. |

Any budget breach causes the pod to exit with a structured failure
report including which budget tripped and a snapshot of progress (files
touched, last validate hints).

## Enforcement

- Wall-clock: a sidecar container with a `sleep 2700 && kubectl delete`
  watchdog, plus a hook `PreToolUse` check.
- Token budgets: hooks inspect the Claude Code session's cumulative
  usage (exposed via environment / transcript).
- Counts: hooks maintain a counter in a file the agent cannot write to.

## Consequences

- Some large refactors won't fit. Those tasks should be split by the
  planner — good pressure on planning quality.
- Budgets will need tuning. The audit log records per-run consumption so
  we can revisit with data.
- A hostile agent cannot avoid budgets because they're enforced by
  hooks, not honor system.

## Alternatives considered

**Token-only budget.** Rejected — doesn't bound wall-clock or cost of
CI runs and external calls.

**Dynamic budgets based on task complexity.** Interesting but
premature; we lack data to set a complexity → budget function.
Revisit after ~100 runs.

**No explicit budgets, just rely on pod timeout.** Rejected — no signal
to the agent about *why* it failed, no opportunity for a graceful
failure report to the planner.
