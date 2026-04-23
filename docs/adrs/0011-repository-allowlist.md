# ADR-0011: Repository allowlist and scoped GitHub App

**Status:** Accepted
**Date:** 2026-04-22

## Context

The implementer modifies code in a target repository, but we must bound
*which* repos and *what scope* of access it has. An implementer that can
read any org repo, or push to any repo, is a much larger compromise
surface than one locked to a single allowlisted repo.

## Decision

- A version-controlled `config/target-repos.yaml` lists approved target
  repos and the required GitHub Actions check suite names per repo.
- The hydrated prompt names the target repo; the MCP server validates
  it against the allowlist before dispatch.
- A single GitHub App (e.g. `bg-agent-implementer`) is installed on
  each allowlisted repo with minimum permissions:
  - `contents: write`
  - `pull_requests: write`
  - `checks: read`
  - No org-level or other repo permissions.
- At dispatch time, the dispatcher mints an installation token scoped
  to **that one repo only** with a lifetime bounded to the run's wall-
  clock budget.
- The planner accesses allowlisted repos via a **separate, read-only**
  GitHub App (`bg-agent-planner`) with `contents: read` only, so the
  planner literally cannot mutate anything even if compromised.

Planner code search options:

- **v1:** local clones on the planner's workspace, read via `ripgrep`
  through an MCP tool. Simple and dependency-free.
- **v2:** Sourcegraph or similar if we need multi-repo search at scale.

## Consequences

- Adding a target repo is a reviewed change to `config/target-repos.yaml`
  (protected by CODEOWNERS per ADR-0005).
- Two GitHub Apps to manage, but cleanly separated by privilege.
- An attacker in a target repo cannot pivot to other repos via the
  implementer — the installation token only grants access to one.

## Alternatives considered

**Single GitHub App with broader scope.** Rejected — violates least
privilege.

**Personal access tokens.** Rejected — not attributable to a service,
hard to rotate, too much standing privilege.

**No allowlist, trust the planner to pick repos.** Rejected — the
planner is a trust boundary; repo scope should be policy, not model
output.
