# ADR-0003: Hook-enforced PR gate, not prompt-enforced

**Status:** Accepted
**Date:** 2026-04-22

## Context

The PR-opening rule — "must pass validate and have green CI" — could be
enforced by telling the implementer in its system prompt, or by wiring
Claude Code hooks that intercept the tool calls.

Prompts are persuasion. Hooks are code. A jailbroken or confused LLM
can violate a prompt; it cannot violate a hook without first breaking the
hook mechanism itself.

## Decision

The PR gate lives in Claude Code hooks, specifically:

- `PreToolUse` on `Bash(git push ...)` — records the pushed SHA.
- `PreToolUse` on `Bash(gh pr create ...)` (or equivalent):
  1. Looks up the current HEAD SHA.
  2. Asserts the audit log has a recent `validate=pass` for that SHA.
  3. Asserts the GitHub API reports all required checks passing for that
     SHA.
  4. Only if both pass, allows the tool call.
- `PostToolUse` on `Bash(validate)` — records the result to the audit
  log via an out-of-pod sink the agent cannot write to directly.

Additionally, `PreToolUse` blocks any `Bash` command not on the
allowlist, and blocks any attempt to modify the hook configuration.

The system prompt can (and should) *describe* the gate so the agent
behaves cooperatively — but the hook is what enforces it.

## Consequences

- Prompt injection cannot bypass the gate.
- Hook code is security-critical and needs its own review rigor (see
  ADR-0005 — hook code is in the allowlist-protected path).
- The agent's "theory of mind" of the gate may drift from reality. We
  mitigate by having the hook return structured error messages when it
  blocks so the agent can self-correct.

## Alternatives considered

**System prompt only.** Rejected — prompt injection bypasses it.

**CI-only gate (PR can open but merge is blocked).** Rejected — an
open PR is already a partial compromise (reviewer fatigue, possible
click-through).

**Policy engine at the K8s layer (admission controller on git push).**
Attractive but complex; revisit in a later phase if the hook approach
proves inadequate.
