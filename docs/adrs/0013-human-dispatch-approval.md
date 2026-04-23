# ADR-0013: Human approval required to dispatch

**Status:** Accepted
**Date:** 2026-04-22

## Context

Dispatch is the highest-privilege action in the system — it spawns code
execution with write access to a real repo. If the planner LLM could
trigger dispatch on its own, prompt injection anywhere in its context
would be a path to PR creation.

## Decision

- Dispatch requires a **signed human approval token** bound to the
  hydrated prompt.
- Approval flow in chat:
  1. Planner drafts a hydrated prompt, posts it to the chat thread.
  2. Human reviews the prompt in-thread.
  3. Human issues `/approve` (or platform equivalent). The chat adapter
     verifies the user's platform identity, then calls the MCP server's
     `dispatch.issue_approval` endpoint with the prompt content hash.
  4. MCP server checks the Discord/Slack/GitHub user ID against the
     `authorized_operators` list, then signs a token:
     ```
     { "prompt_hash": "...", "operator": "...", "iat": ..., "exp": iat+600, "jti": "..." }
     ```
  5. The token is presented by the planner to `dispatch.request`. The
     MCP server verifies signature, expiry, one-time-use (`jti`
     tracked), and that `prompt_hash` matches the prompt being
     dispatched.
- Tokens are valid for 10 minutes, single-use, bound to the prompt
  content.
- The identity of the approving human, the prompt hash, and the token
  `jti` are written to the audit log **before** the pod is launched.

**For forensic purposes:** approval is always a human action, always
logged, always bound to a specific prompt. A compromised planner cannot
dispatch without a fresh human approval.

## Consequences

- Every dispatch has a named human accountable for it.
- Humans must read the hydrated prompt before approving. This is work,
  but it is the point — the plan has to be legible enough to approve.
- Large prompts are harder to review. Mitigation: planner renders a
  structured summary alongside the raw prompt.
- A compromised chat-platform account is still a risk. We partially
  mitigate by requiring MFA on operator accounts (platform-level
  policy, not enforced by us).

## Alternatives considered

**Auto-dispatch when the planner's confidence is high.** Rejected —
removes the human from the highest-privilege transition.

**Approval by any org member.** Rejected — `authorized_operators`
list is small and reviewed.

**Approval via a separate web UI.** Considered; chat-based approval is
simpler for v1 and keeps the forensic record in the same medium as
the conversation.
