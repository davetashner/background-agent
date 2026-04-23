# ADR-0012: Binary outcome for implementer runs

**Status:** Accepted
**Date:** 2026-04-22

## Context

An implementer could end a run in many states: partial commit, stashed
work, half-written PR, "I'll finish later" notes, etc. Every
intermediate state is a security and operational liability:

- Partial state persists across runs (violates ADR-0006's ephemerality).
- Partial commits land on feature branches and confuse future runs.
- A "I'll finish later" hint encourages humans to relax budgets.

## Decision

An implementer run ends in exactly one of two ways:

1. **PR opened.** Branch pushed, CI green, `validate` pass recorded,
   PR URL reported back to the planner.
2. **No PR.** The pod exits with a structured failure report:
   ```json
   {
     "outcome": "no_pr",
     "reason": "validate_failed_after_retries | ci_failed | budget_exhausted | tool_blocked | internal_error",
     "details": "...",
     "suggestions": ["clarify X in the prompt", "add Y to the allowlist", ...],
     "last_validate_hints": [...]
   }
   ```

No intermediate state survives the pod. No partial branches are left
behind (pushed branches without a PR are cleaned up by the dispatcher).

The planner receives the failure report and can relay suggestions to
the human in chat.

## Consequences

- Clean mental model: a run produces a PR or it doesn't.
- Retries are always from scratch, with a potentially refined prompt.
  Human is in the loop for each attempt.
- Some half-done work that would be salvageable is thrown away. We
  accept this to keep the security surface small.

## Alternatives considered

**Allow draft PRs as a third outcome.** Rejected — draft PRs are still
PRs; reviewer discipline erodes.

**Persist partial state in a "work-in-progress" volume for future
runs.** Rejected — cross-run state is exactly what ADR-0006 forbids.

**Report progress via chat without opening a PR.** Rejected —
conflates "progress update" with "dispatched work"; easier to reason
about if runs are opaque work units.
