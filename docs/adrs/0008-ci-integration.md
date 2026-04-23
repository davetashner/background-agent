# ADR-0008: GitHub Actions for CI in v1, alternatives documented

**Status:** Accepted
**Date:** 2026-04-22

## Context

The implementer must prove CI passes on the proposed commit before
opening a PR. CI must run somewhere trustworthy — a runner the agent
can't influence.

User wants to start with GitHub Actions for the prototype but
documented alternatives for later (in-pod runner, Jenkins, etc.).

## Decision for v1

- Target repos already use or will adopt GitHub Actions.
- Implementer pushes to a `background-agent/<run-id>` branch on the
  target repo. GitHub Actions runs configured workflows on that push.
- Hook on `Bash(gh pr create)` polls the GitHub checks API for the
  pushed commit SHA:
  - Wait up to 10 minutes (configurable per run).
  - Only the set of check suites configured as "required" on the target
    branch must pass.
  - If any required check is `failure`/`cancelled`/`timed_out`, block
    the PR and return the check names (not logs) to the agent for retry.
- Runners: GitHub-hosted runners for the prototype. Self-hosted runners
  are out of scope for v1 (they'd need their own isolation story).

## Alternatives documented (for later phases)

### In-pod CI runner

Run the build+test inside the implementer pod itself (or a sidecar).

- **Pro:** fast feedback; no external dependency; easy to reproduce
  locally.
- **Con:** agent can influence the CI environment (install packages,
  modify test commands). Undermines the "independent verification"
  goal. Could be salvaged by making the in-pod runner a separate
  container with its own immutable image and a read-only bind mount of
  the working tree.
- **When to reconsider:** if external CI latency becomes the dominant
  loop cost.

### Self-hosted GitHub Actions runners

Same API, but runners live on our infrastructure.

- **Pro:** network isolation control; access to internal resources for
  tests.
- **Con:** runner security model is its own project. Ephemeral runners
  per job are doable but non-trivial.
- **When to reconsider:** when the target repo's tests need internal
  network access.

### Jenkins / Buildkite / CircleCI

External dedicated CI.

- **Pro:** mature runner isolation; more flexible policies.
- **Con:** more moving parts; another credential surface for the hook
  to authenticate with.
- **When to reconsider:** if we need build policies GitHub Actions
  can't express (e.g., multi-arch matrixes with approval gates).

## Consequences

- v1 works only for repos on GitHub with Actions enabled.
- Required-check configuration in each target repo becomes part of
  onboarding.
- The poll timeout needs tuning; long-running test suites may need
  higher budgets per repo.

## Alternatives considered (and rejected for v1)

**Trust the agent's claim that CI passed.** Rejected — prompt
injection bypasses it.

**Skip CI for v1, rely on validate only.** Rejected — validate is
opaque and cannot cover all correctness checks; CI is the
project-authoritative signal.
