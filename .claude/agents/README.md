# Agent team

Claude Code subagents purpose-built for this project. Each persona has
a narrow remit, a clear set of tools, and is expected to hand off to
other personas when the work leaves its domain.

## Team roster

| Agent | When to use |
|---|---|
| `security-reviewer` | Any change touching auth, IAM, hooks, allowlist, audit log, validate, dispatcher, NetworkPolicies. Proactively on diffs that cross trust boundaries. |
| `adr-author` | Non-obvious design decisions that need capturing. Supersedes or amends prior ADRs. |
| `k8s-platform-engineer` | AWS, EKS, Terraform, Helm, IRSA, NetworkPolicies, gVisor, cross-account wiring. |
| `claude-code-hook-engineer` | Claude Code hooks for PR gate, allowlist enforcement, validate recording, budgets. |
| `agent-systems-engineer` | MCP server, planner, dispatcher, chat adapters, Bedrock integration. |
| `validate-tool-engineer` | Validate service and the hint abstraction layer. |
| `backlog-curator` | Beads grooming, stale work, closure reconciliation. |

## Handoff patterns

Most work touches two or three domains. Typical chains:

- New IAM policy → `k8s-platform-engineer` → `security-reviewer`
- New MCP tool → `agent-systems-engineer` → `security-reviewer`
- New hook → `claude-code-hook-engineer` → `security-reviewer`
- New validate checker → `validate-tool-engineer` → `security-reviewer`
- Cross-service design tradeoff → whoever noticed → `adr-author`

`security-reviewer` is the terminal reviewer for anything touching
trust boundaries. It does not write code; it reviews and requests
changes.

## What's intentionally missing

- **Planner engineer** / **dispatcher engineer** / **chat adapter
  engineer** — these collapse into `agent-systems-engineer` because the
  LLM-app layer is tightly coupled and a single persona avoids
  duplicated context.
- **Threat modeler** — folded into `security-reviewer`. One reviewer is
  enough; splitting would fragment the threat model's custody.
- **Python / Go / TypeScript specialist** — language expertise comes
  from the task, not the persona. Agents pick up the language of the
  file they're editing.

If the team composition feels wrong after a few weeks of use,
revise it with an ADR.
