# ADR-0010: Project tracking via beads

**Status:** Proposed
**Date:** 2026-04-22

## Context

We need an issue/task tracker for the project. Options considered:

- **beads** — file-based, lives in the repo, CLI-driven, works natively
  with worktrees and branches.
- **Linear / Jira / GitHub Issues** — hosted, richer UI, but external.
- **Plain markdown task lists in the repo** — simplest, but no IDs, no
  dependencies, no status queries.

Given this project is security-sensitive and spec-driven, tracker state
changing in lockstep with code via PRs is desirable. That favors a
file-based tracker.

## Decision

Use beads as the project tracker.

- Initialize in `.beads/` at the repo root.
- Every PR that implements a tracked task references the bead ID in its
  commits and `Closes` lines (per the user's global CLAUDE.md).
- The seed backlog lives in `docs/backlog-seed.md` and is loaded into
  beads by the user once they approve; agents should not `beads init`
  or import without explicit sign-off to avoid creating noise.
- Epics map to phases in `implementation-plan.md`.

## Consequences

- Task state is reviewable like code.
- Offline work is possible.
- New contributors need beads installed; onboarding friction is small
  and documented.
- No built-in web UI for stakeholders; mitigated by generating a
  markdown "status" snapshot on demand.

## Alternatives considered

**GitHub Issues.** Good integration with PRs, but status queries over
many issues are harder to reason about from the CLI, and issue state is
a separate thing to keep in sync.

**Plain markdown.** Too loose for a multi-phase project with
dependencies.

**Linear.** Nice UI, but another account boundary and an external
system for something that benefits from living with the code.
