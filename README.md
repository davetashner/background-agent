# background-agent

A prototype of a **security-first background coding agent** inspired by Spotify's
background coding agent write-ups ([part 1][p1], [part 2][p2], [part 3][p3]).

The goal is an agent that can take a natural-language task from a human in chat,
develop a plan collaboratively, and then have a sandboxed implementer agent
produce a pull request for human review — with hard guardrails against the
implementer bypassing review, exfiltrating data, or running unbounded.

[p1]: https://engineering.atspotify.com/2025/11/spotifys-background-coding-agent-part-1
[p2]: https://engineering.atspotify.com/2025/11/context-engineering-background-coding-agents-part-2
[p3]: https://engineering.atspotify.com/2025/12/feedback-loops-background-coding-agents-part-3

## Components

- **Chat adapters** — thin frontends for Discord (primary), Slack, and GitHub
  comments. They do no LLM work; they forward messages to the planner.
- **Planner service** — the conversational LLM that interviews the human,
  researches code, and produces a "fully hydrated" implementation prompt. It
  can read code and use tools, but **cannot commit or open PRs**.
- **MCP tool server** — front-door for the planner's tools (code search, file
  read, dispatch-to-implementer, audit logging). Because multiple chat clients
  may feed the planner, tool access is consolidated here.
- **Dispatcher** — receives approved plans from the planner and launches a
  fresh Kubernetes pod per implementation request.
- **Implementer agent** — Claude Code running in a locked-down pod. One pod
  per unique prompt. Accepts input *only* from the dispatcher, never from
  humans directly. Exits with exactly two outcomes:
  1. A PR is opened for human review.
  2. No PR is opened; a structured failure report is returned to the planner.
- **Validate tool** — an opaque verifier the implementer must pass before it
  is allowed (via Claude Code hooks) to open a PR. The implementer does not
  know what `validate` checks.
- **Audit log** — every system prompt sent from planner to implementer, every
  tool call, and every validate result is persisted for later review.

## Security model

- **Two-human tool allowlist.** The set of bash commands and tools the
  implementer may invoke lives in a version-controlled allowlist. Changes
  require a PR approved by two humans (enforced via GitHub branch protection
  and CODEOWNERS).
- **Hook-enforced PR gate.** Claude Code hooks block `PR open` actions unless
  (a) `validate` has passed on the current commit **and** (b) CI has reported
  green on the pushed branch.
- **No human → implementer channel.** The implementer pod has no inbound
  ingress. It pulls its prompt from the dispatcher at startup and exits.
- **Planner cannot write.** The planner's credentials grant read-only git
  access and no PR/commit scopes.
- **Per-pod ephemeral sandboxing.** Each implementer run gets a fresh
  Kubernetes namespace with egress restricted to a pinned allowlist (git
  host, CI API, validate service, audit sink).
- **Loop budgets.** Wall-clock, token, and validate-call budgets are enforced
  by hooks; the implementer cannot spin forever.

## Flow

```
human  ──▶ chat adapter ──▶ planner (LLM) ──▶ hydrated prompt
                                   │
                                   ▼
human approves in chat ──▶ dispatcher ──▶ K8s pod (implementer)
                                                  │
                                                  ├─▶ validate
                                                  ├─▶ push branch + CI
                                                  └─▶ open PR (hook-gated)
                                                          │
                                                          ▼
                                                  human reviews PR
```

## Status

Prototype — design phase. See `CLAUDE.md` for working agreements and the
`docs/` directory (once created) for the design doc and threat model.
