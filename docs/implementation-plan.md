# Implementation plan

Phased plan from empty repo to a production-shaped prototype on AWS EKS,
organized into **waves** of parallelizable work.

A *wave* is a set of tasks that can be attempted at the same time
because their dependencies have all landed in previous waves. Inside a
wave, tasks that can run in parallel across multiple workers are marked
**‖ parallel**; tasks that must run sequentially are **→ serial**.

Every task below maps to a beads issue — use `bd show <id>` for
acceptance criteria and current status. Phase-level epic IDs appear in
parentheses next to each phase heading.

---

## Phase 0 — Planning  (epic `background-agent-jr0`)

**Exit criterion:** design artifacts merged, beads seeded, user sign-off.
**Status:** complete.

### Wave 0.1 — Docs (‖ parallel)
- README and CLAUDE.md
- Design doc, threat model, ADRs 0001–0014
- Phased implementation plan (this doc)
- Backlog seed

### Wave 0.2 — Tracker and remote (‖ parallel)
- `bd init` and seed backlog
- Push `main` to origin, apply branch protection

---

## Phase 0.5 — AWS organization baseline  (epic `background-agent-ofj`)

**Exit criterion:** CloudTrail, GuardDuty, Config, SCPs, budgets, and
named IAM roles live across the 3 starter accounts. No workloads can
run safely before this phase completes.

**Blocks:** phases 1, 5, 6.

### Wave 0.5.1 — Decision ADRs (‖ parallel)
- `ofj.1` ADR-0015 IAM role design — **landed in this PR**
- `ofj.2` ADR-0016 Bedrock for LLM access — **landed in this PR**

### Wave 0.5.2 — Foundational terraform (‖ parallel)
Independent modules — 4 workers can run concurrently:

- `ofj.3` CloudTrail org trail to security account
- `ofj.4` GuardDuty org-wide with security as delegated admin
- `ofj.5` AWS Config aggregator in security account
- `ofj.6` SCPs: root-user denial, region denial, IAM-user denial
- `ofj.7` Budget alarms per account

### Wave 0.5.3 — IAM roles (→ serial, blocked by 0.5.1)
- `ofj.8` Named IAM roles per account (Admin, DeployOps, ReadOnly,
  AuditRead, AuditWrite) per ADR-0015

---

## Phase 1 — Local walking skeleton  (epic `background-agent-ul7`)

**Exit criterion:** on `kind`, a human can drive CLI mock → planner →
approve → pod → PR against a toy fixture repo. PR gate, validate, and
CI are stubbed.

**Blocked by:** Phase 0.5 (for clean workstation auth).

### Wave 1.1 — Foundation decisions (→ serial)
- `ul7.1` Decide service languages and monorepo layout — the
  critical-path unblocker for everything downstream.

### Wave 1.2 — Parallel foundations (‖ parallel, blocked by 1.1)
- `ul7.2` Toy target repo fixture (no deps — can start at any time)
- `ul7.5` MCP tool server v1 — first because planner and dispatcher
  both depend on it

### Wave 1.3 — Services (‖ parallel, blocked by 1.2)
- `ul7.3` CLI mock chat adapter
- `ul7.4` Planner service skeleton (needs MCP)
- `ul7.6` Dispatcher targeting kind (needs MCP)
- `ul7.7` Implementer pod base image

### Wave 1.4 — Integration (→ serial, blocked by all of 1.3)
- `ul7.8` Phase 1 end-to-end smoke

**Parallelism note:** with 3 workers, phase 1 can complete in roughly
the length of its longest single-task chain:
1.1 → 1.2 → 1.3 → 1.4.

---

## Phase 2 — Discord adapter + real planner dialogue  (epic `background-agent-pqk`)

**Exit criterion:** a human in a private Discord channel can drive the
phase-1 flow end-to-end.

### Wave 2.1 — Adapter components (‖ parallel)
- `pqk.1` Discord bot scaffolding and session state
- `pqk.3` Operator allowlist and identity verification

### Wave 2.2 — Approval flow (→ serial, blocked by 2.1)
- `pqk.2` /approve slash command with signed tokens

### Wave 2.3 — Integration (→ serial, blocked by 2.2)
- `pqk.4` Phase 2 end-to-end in private Discord

---

## Phase 3 — Hook-enforced PR gate + opaque validate  (epic `background-agent-12a`)

**Exit criterion:** adversarial prompts cannot open a PR; honest path
still works.

### Wave 3.1 — Hook framework (→ serial)
- `12a.1` Hook framework integration

### Wave 3.2 — Hooks (‖ parallel, blocked by 3.1)
- `12a.2` PreToolUse allowlist enforcement
- `12a.3` PreToolUse PR gate (may be stubbed against 3.1 initially)
- `12a.4` PostToolUse validate recorder

### Wave 3.3 — Validate service (→ serial, can start during wave 3.1)
- `12a.5` Validate service with abstraction layer

### Wave 3.4 — Red team (→ serial, blocked by 3.2 and 3.3)
- `12a.6` Red-team test suite for PR gate

---

## Phase 4 — GitHub Actions CI integration  (epic `background-agent-o3p`)

### Wave 4.1 — Setup (‖ parallel)
- `o3p.1` Configure target repo required checks
- `o3p.2` Hook: poll GitHub checks API for pushed SHA

### Wave 4.2 — Integration (→ serial, blocked by 4.1)
- `o3p.3` CI failure reporting back to implementer
- `o3p.4` Smoke test: intentional breakage blocks PR

---

## Phase 5 — Audit logging hardening  (epic `background-agent-aop`)

**Blocked by:** Phase 0.5 (CloudTrail + security-account bucket).

### Wave 5.1 — Bucket and IAM (‖ parallel)
- `aop.3` S3 Object Lock bucket configuration
- `aop.2` Cross-account IAM for audit writes
- `aop.4` Secret and credential scrubber (can be authored standalone)

### Wave 5.2 — Shipper (→ serial, blocked by 5.1)
- `aop.1` Audit log shipper service

### Wave 5.3 — Indexing (→ serial, blocked by 5.2)
- `aop.5` Per-run audit index documents

---

## Phase 6 — AWS EKS deployment  (epic `background-agent-920`)

**Blocked by:** Phases 0.5 and 5.

### Wave 6.1 — Terraform foundations (‖ parallel)
- `920.1` Terraform: AWS Organizations OU structure refinements
- `920.2` Terraform: EKS clusters per env
- `920.3` IRSA roles per service

### Wave 6.2 — App delivery (‖ parallel, blocked by 6.1)
- `920.4` Helm charts for each service
- `920.5` NetworkPolicies: default-deny
- `920.6` gVisor runtime class + node group

### Wave 6.3 — Cross-account integration (→ serial, blocked by 6.2)
- `920.7` Cross-account dispatch wiring

---

## Phase 7 — Hardening and observability  (epic `background-agent-70a`)

### Wave 7.1 — Observability (‖ parallel)
- `70a.1` GuardDuty findings pipeline + VPC Flow Logs
- `70a.3` Load test: 10 concurrent implementer pods

### Wave 7.2 — Docs and decisions (‖ parallel)
- `70a.2` Runbooks
- `70a.4` Evaluate gVisor vs Fargate

### Wave 7.3 — Sign-off (→ serial)
- `70a.5` Residual threat model review

---

## Always-parallelizable work

These tasks have no phase dependency and any available worker can take
them at any time:

- `cfe` Pre-commit hooks (secret scanning)
- `4pf` Agent team personas in `.claude/agents/`
- `9xr` CODEOWNERS
- `rmm` config/implementer-tools.yaml
- `rv9` config/target-repos.yaml
- `h2g` Rolling open-questions tracker for ADRs 0004 and 0009

## Worker/wave scaling guide

| Phase | Sweet spot | Reason |
|---|---|---|
| 0.5 | 3–4 | Wave 0.5.2 has 5 independent terraform modules |
| 1 | 3 | Wave 1.3 has 4 services |
| 2 | 2 | Limited parallelism |
| 3 | 3 | Waves 3.2 and 3.3 are parallel |
| 4 | 2 | Mostly sequential |
| 5 | 3 | Wave 5.1 three-way parallel |
| 6 | 3–4 | Waves 6.1 and 6.2 have multiple modules |
| 7 | 2–3 | Mix of docs and load-testing |

## Out of prototype scope

- Multi-tenant isolation (per-team namespaces, per-team operator lists)
- Plan templates / memory of past plans
- Agent-to-agent orchestration beyond planner → implementer
- Self-service onboarding of new target repos by non-security-team users
- Staging and production account pairs (`bg-agent-stg-*`,
  `bg-agent-prd-*`)
