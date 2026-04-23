# Implementation plan

Phased plan from empty repo to a production-shaped prototype on AWS EKS.
Each phase has an exit criterion the user can demo. No phase overlaps in
scope — finish one before starting the next.

## Phase 0 — Planning (current)

**Exit criterion:** design doc, threat model, ADRs, and seeded backlog all
merged to `main`.

- [x] README and CLAUDE.md.
- [x] Design doc.
- [x] Threat model.
- [x] ADRs for every load-bearing decision.
- [ ] Backlog seeded (in beads once tracker is initialized).
- [ ] User sign-off on phase 0.

## Phase 1 — Local walking skeleton

**Exit criterion:** on a developer laptop with `kind`, a human can chat with
a Discord-mocked CLI, get a plan, approve it, and see a pod run Claude Code
against a toy repo — but the PR gate, validate, and CI are all stubbed to
always pass. The purpose is to prove the control flow.

- Monorepo layout with a package per component.
- Chat adapter: CLI-mock (Discord adapter comes in phase 2). The mock
  reads/writes stdin/stdout so we can run the flow without network.
- Planner: thin wrapper around an LLM with a fixed system prompt.
- MCP tool server: `code.search` over local clones, `code.read`,
  `plan.submit`, `dispatch.request` (stub), `audit.log` (writes JSONL).
- Dispatcher: a process that receives a plan and `kubectl apply`s a Job
  manifest.
- Implementer Job: runs Claude Code with hardcoded allowlist and stubbed
  validate that always returns `{ok: true}`.
- Toy repo: a small Python repo in `fixtures/toy-repo/` with a few simple
  functions; used as the dispatch target.

## Phase 2 — Discord adapter + real planner dialogue

**Exit criterion:** a human in a private Discord channel can drive the same
flow as phase 1.

- Discord adapter: deploy as its own service, authenticate via bot token.
- Session management: planner state keyed by Discord thread ID.
- Approval UX: a `/approve` slash command that signs the current hydrated
  prompt and returns a token to the MCP server.
- Operator allowlist: Discord user IDs in a config file, loaded by MCP
  server.

## Phase 3 — Hook-enforced PR gate + opaque validate

**Exit criterion:** a malicious implementer prompt ("open a PR without
running validate") fails; the honest path still works.

- Implement Claude Code hooks:
  - `PreToolUse` for `Bash(git push)` and `Bash(gh pr create)` checks
    audit log for a passing validate record on the current commit SHA.
  - `PostToolUse` for `Bash(validate)` records the result to audit log.
  - `PreToolUse` blocks any command not on the allowlist.
- Validate service: runs as a sidecar or separate Service; implementer
  calls via HTTP on localhost.
- Abstraction layer in validate service that maps tool-specific errors
  to generic hints (ADR-0004).
- Red-team tests: a suite of adversarial prompts that the hook must block.

## Phase 4 — CI integration (GitHub Actions)

**Exit criterion:** PR opens only when CI is green; hook verifies via
GitHub API, not agent output.

- Configure target repo with required status checks on feature branches.
- Implementer pushes to a `background-agent/<run-id>` branch.
- Hook on `Bash(gh pr create)` polls check run status for the pushed SHA.
- 10-minute timeout. Exceeding the timeout = failure report back to
  planner.

## Phase 5 — Audit logging hardening

**Exit criterion:** a third party can reconstruct every action of every run
from the audit log alone.

- Move audit sink from JSONL-on-disk to S3 with Object Lock.
- Ship every MCP call, every planner message, every implementer tool call.
- Add per-run index documents for fast querying.
- Document retention policy.

## Phase 6 — AWS EKS deployment

**Exit criterion:** the system runs on the topology from ADR-0007.

- Terraform for AWS Organizations accounts, EKS clusters, IRSA roles.
- Helm charts for each service.
- NetworkPolicies: default-deny, explicit allow per service.
- Implementer pods run on a dedicated node group with gVisor runtime class.
- ECR with image signing (Sigstore).

## Phase 7 — Hardening and observability

**Exit criterion:** threat-model residual risks are addressed or
explicitly accepted.

- Egress monitoring (VPC flow logs to the security account).
- GuardDuty for runtime detection.
- Runbooks for: stuck pod, compromised token, allowlist drift.
- Load test: 10 concurrent implementer pods.

## Out of prototype scope (noted for the roadmap)

- Multi-tenant isolation (per-team namespaces, per-team operator lists).
- Plan templates / memory of past plans.
- Agent-to-agent orchestration beyond planner → implementer.
- Self-service onboarding of new target repos by non-security-team users.
