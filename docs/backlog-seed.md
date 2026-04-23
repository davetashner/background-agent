# Backlog seed

Ready-to-load into beads once you approve ADR-0010 and run `bd init`.
Each block below is one issue. Epics correspond to phases in
`implementation-plan.md`.

Shape roughly matches beads CLI flags (`--title`, `--body`, `--type`,
`--parent`, `--blockedBy`). Treat titles as suggestions.

---

## Epic: Phase 0 — Planning

### E0: Ship design documentation

**type:** epic
**body:** Produce design doc, threat model, ADRs, implementation plan,
and seed backlog. Phase 0 exit criterion in `implementation-plan.md`.

- [x] README + CLAUDE.md.
- [x] design.md.
- [x] threat-model.md.
- [x] ADRs 0001–0014.
- [x] implementation-plan.md.
- [ ] User sign-off on phase 0 artifacts.
- [ ] Initialize beads and load seed backlog.

---

## Epic: Phase 1 — Local walking skeleton

### E1: Stand up a local walking skeleton

**type:** epic
**body:** End-to-end flow on a laptop using `kind`. PR gate, validate,
and CI are stubbed; focus is control flow, not security.

Blocked by: E0 sign-off.

#### Stories under E1

**S1.1 — Monorepo layout and language choice**
- Decide language for each service (planner, MCP server, dispatcher,
  chat adapter) and document in an ADR.
- Create workspace skeleton (no implementation yet).

**S1.2 — Toy target repo fixture**
- A small Python repo under `fixtures/toy-repo/` with 2–3 functions
  and a trivial test suite.
- Serves as the default dispatch target for phase 1.

**S1.3 — CLI chat adapter (mock Discord)**
- Reads stdin, writes stdout.
- Emits normalized `ChatMessage` envelopes to the planner.
- `/approve` command in the CLI.

**S1.4 — Planner service skeleton**
- Thin LLM wrapper with a fixed system prompt describing the task.
- Connects to MCP tool server.
- Produces a hydrated prompt artifact.

**S1.5 — MCP tool server**
- `code.search`, `code.read`, `repo.list_allowlisted`.
- `plan.submit`, `dispatch.request` (stub — logs but does not dispatch).
- `audit.log` writes to local JSONL.

**S1.6 — Dispatcher (kind-targeted)**
- Accepts a hydrated prompt + approval token.
- Creates a K8s Job that runs Claude Code with the prompt.
- Streams logs back.

**S1.7 — Implementer pod image**
- Dockerfile with Claude Code, a stubbed `validate` that always passes,
  git, and the minimal allowlist tools.
- Startup script that reads the prompt from a mounted file and exits
  after one session.

**S1.8 — End-to-end smoke**
- Script that runs the full flow against the toy repo.
- Produces a PR (against a local git server or a disposable GitHub
  test repo — decide).

---

## Epic: Phase 2 — Discord adapter + real planner dialogue

### E2: Discord adapter with approvals

Blocked by: E1 smoke passing.

- **S2.1** Discord bot scaffolding, bot token loading, channel/guild
  config.
- **S2.2** Session state keyed by thread ID.
- **S2.3** `/approve` slash command that calls MCP server to issue
  signed token.
- **S2.4** Operator allowlist file + verification.
- **S2.5** End-to-end in a private Discord server.

---

## Epic: Phase 3 — Hook-enforced PR gate and opaque validate

### E3: Security gate

Blocked by: E2.

- **S3.1** Hook framework — load hooks into implementer session,
  configurable per run.
- **S3.2** `PreToolUse` allowlist enforcement.
- **S3.3** `PreToolUse` on git push and pr create; verifies audit log.
- **S3.4** `PostToolUse` on validate; writes result to audit log via
  out-of-pod sink.
- **S3.5** Validate service — separate container; abstraction layer that
  maps tool output to generic hints (ADR-0004).
- **S3.6** Red-team test suite of adversarial prompts.

---

## Epic: Phase 4 — GitHub Actions integration

### E4: CI gate

Blocked by: E3.

- **S4.1** Configure target repo with required checks.
- **S4.2** Hook that polls GitHub checks API for pushed SHA.
- **S4.3** Timeout and reporting on CI failures back to the implementer.
- **S4.4** Smoke test: intentional breakage → PR blocked.

---

## Epic: Phase 5 — Audit logging

Blocked by: E4.

- **S5.1** Shipper service to S3.
- **S5.2** Cross-account role to write into security-account bucket.
- **S5.3** Object Lock configuration.
- **S5.4** Secret/credential scrubber.
- **S5.5** Per-run index documents.

---

## Epic: Phase 6 — AWS EKS deployment

Blocked by: E5.

- **S6.1** Terraform: AWS Organizations, accounts, OU structure.
- **S6.2** Terraform: EKS clusters (ctrl and impl) per env.
- **S6.3** IRSA roles for each service.
- **S6.4** Helm charts.
- **S6.5** NetworkPolicies (default-deny + explicit allow).
- **S6.6** gVisor runtime class + dedicated node group.
- **S6.7** Cross-account dispatch wiring.

---

## Epic: Phase 7 — Hardening

Blocked by: E6.

- **S7.1** GuardDuty + VPC Flow Logs wiring.
- **S7.2** Runbooks: stuck pod, compromised token, allowlist drift.
- **S7.3** Load test (10 concurrent implementer pods).
- **S7.4** Revisit gVisor vs. Fargate.
- **S7.5** Residual threat-model review and sign-off.

---

## Standalone issues (not under an epic)

- **T-config-repos** Create `config/target-repos.yaml` schema and
  starter content.
- **T-config-tools** Create `config/implementer-tools.yaml` with
  minimal starter allowlist and CODEOWNERS entry.
- **T-codeowners** Write `CODEOWNERS` requiring 2 reviews on
  `config/**`, `hooks/**`, `services/validate/**`, `services/dispatcher/**`.
- **T-security-adr-followups** Track open questions from ADRs 0004 and
  0009 as they accumulate data.
