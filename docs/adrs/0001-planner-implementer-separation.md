# ADR-0001: Separation of planner and implementer agents

**Status:** Accepted
**Date:** 2026-04-22

## Context

An LLM that both converses with humans and writes code has a large attack
surface: prompt injection via chat or repo content can escalate into commit
or PR actions. Even without adversaries, blending planning and execution
makes it hard to audit which parts of an LLM's context produced which code
change.

## Decision

Split the system into two agents with distinct trust levels and
capabilities:

- **Planner** — converses with humans, reads code, produces plans. Has
  **no** ability to commit, push, or open PRs. Long-lived session.
- **Implementer** — reads only a hydrated prompt, writes code, opens PRs.
  Has **no** ability to converse with humans or accept mid-run input.
  Short-lived, one-shot pod.

The handoff between them is a single immutable artifact (the hydrated
prompt + signed human approval).

## Consequences

- Prompt injection into the planner cannot directly produce a PR.
- Audit trail is clear: one prompt = one pod = one PR attempt.
- We pay the cost of running two agents and transferring context
  explicitly between them. Context hydration is the planner's main job,
  per ADR-0002 and the Spotify part-2 post.
- Features that want a single conversational agent "that does everything"
  are explicitly off the table.

## Alternatives considered

**Single multi-role agent.** Rejected — blending trust levels in one
context window defeats the security goal.

**Planner with constrained tools but still able to dispatch directly.**
Rejected — a compromised planner can still trigger runs. The human
approval gate (ADR-0013) is what breaks this.
