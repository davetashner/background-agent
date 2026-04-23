# Architecture Decision Records

Each ADR captures one non-obvious decision: the context, the options
considered, what we chose, and what we're giving up. Small and focused.

New ADRs keep going up in number; never renumber. Supersede an old ADR by
writing a new one and linking back.

## Index

| # | Title | Status |
|---|-------|--------|
| 0001 | Separation of planner and implementer agents | Accepted |
| 0002 | MCP-fronted tool plane for multi-client planners | Accepted |
| 0003 | Hook-enforced PR gate, not prompt-enforced | Accepted |
| 0004 | `validate` is opaque to the implementer | Accepted |
| 0005 | Tool allowlist requires two-human approval | Accepted |
| 0006 | Per-run ephemeral Kubernetes namespaces | Accepted |
| 0007 | AWS EKS topology | Accepted |
| 0008 | GitHub Actions for CI in v1, alternatives documented | Accepted |
| 0009 | Loop budgets for implementer runs | Accepted |
| 0010 | Project tracking via beads | Proposed |
| 0011 | Repository allowlist and scoped GitHub App | Accepted |
| 0012 | Binary outcome for implementer runs | Accepted |
| 0013 | Human approval required to dispatch | Accepted |
| 0014 | Append-only audit log in S3 Object Lock | Accepted |

## Template

```markdown
# ADR-NNNN: Short title

**Status:** Proposed | Accepted | Superseded by ADR-XXXX
**Date:** YYYY-MM-DD

## Context
Why are we deciding this now? What forces are in play?

## Decision
What we're doing.

## Consequences
What we gain, what we give up, what becomes harder.

## Alternatives considered
Briefly — one paragraph each.
```
