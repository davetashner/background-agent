# ADR-0005: Tool allowlist requires two-human approval

**Status:** Accepted
**Date:** 2026-04-22

## Context

The implementer's capability is bounded by its tool allowlist — the list
of shell commands and Claude Code tools it may use. A silent expansion
of this list (adding `curl`, `ssh`, `aws`, etc.) would dramatically
change the security posture without any PR surface area in the target
repo.

## Decision

- The allowlist lives at `config/implementer-tools.yaml` in this repo.
- The path is owned by CODEOWNERS requiring **two approvals** from the
  `@security-reviewers` group (to be defined when the team exists).
- Branch protection on `main` enforces:
  - Required reviews from CODEOWNERS: 2.
  - No self-review.
  - No force push.
  - Signed commits required.
- The implementer pod reads the allowlist from a read-only ConfigMap
  derived from `main` at deploy time.
- At pod start, the effective allowlist hash is recorded to the audit
  log. A CronJob compares deployed hashes against the latest `main` and
  alerts on drift.

The same CODEOWNERS protection applies to:

- Hook configuration.
- The validate service's abstraction layer.
- Dispatcher code paths that mint credentials.

## Consequences

- Changes to capability are slower by design.
- Human reviewers are in the loop for every capability change — this is
  a feature, not a bug.
- Emergency additions (say, a new linter) need two people available.
  Acceptable trade-off.

## Alternatives considered

**Single approver + audit-after-the-fact.** Rejected — one compromised
account is enough to escalate.

**Allowlist stored in the target repo alongside code.** Rejected — mixes
security policy with application code; target repos have different
review cultures.

**Dynamic allowlist negotiated per run.** Rejected — a negotiable
allowlist is no allowlist at all; it moves the attack surface to the
negotiation protocol.
