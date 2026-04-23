# Design documentation

This directory is the source of truth for the prototype's design. Code and
scripts are intentionally absent until the design stabilizes.

## Layout

- `design.md` — overall architecture and component responsibilities.
- `threat-model.md` — assets, threats, and mitigations.
- `implementation-plan.md` — phased plan from scaffolding to EKS deploy.
- `adrs/` — architecture decision records. See `adrs/README.md` for the index.
- `backlog-seed.md` — initial backlog, ready to load into beads once the
  tracker is initialized.

## Reading order for a new contributor

1. `../README.md` — what this project is.
2. `../CLAUDE.md` — core invariants. Read before changing anything.
3. `design.md` — how the pieces fit.
4. `threat-model.md` — why the pieces fit that way.
5. `adrs/` — the reasoning behind each non-obvious choice.
6. `implementation-plan.md` — what to build next.
