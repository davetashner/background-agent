# Threat model

Scope: the background-agent prototype as described in `design.md`. This is a
working document — expand as new threats surface during implementation.

## Assets

| Asset | Sensitivity |
|-------|-------------|
| Target source repositories | High — production code. |
| GitHub App private key | Critical — grants repo write. |
| Per-run GitHub installation tokens | High — scoped, short-lived. |
| Audit log | High — regulatory and forensic value. |
| Tool allowlist file | High — defines implementer capability. |
| `validate` tool source | Medium — secrecy supports ADR-0004. |
| Planner conversation history | Medium — may contain proprietary context. |
| Hydrated prompts | Medium — reveal internal thinking about systems. |

## Trust zones

```
[internet] --> [chat platform] --> [chat adapter]
                                       |
                                    [MCP tool server] --- [planner LLM]
                                       |
                                    [dispatcher]
                                       |
                                    [implementer pod]  (least-trusted)
```

The implementer pod is treated as the least-trusted component because its
context contains model-generated code that may exhibit prompt-injection or
unintended behavior. All security controls push outward from this pod.

## Threat catalog

### T1 — Planner is prompt-injected to run malicious code

**Vector:** a human (or content in a repo the planner reads) includes a
crafted instruction that tries to get the planner to exfiltrate code or
trigger a malicious dispatch.

**Mitigations:**
- Planner has no commit/push credentials (design invariant).
- `dispatch.request` requires a signed human approval token issued within
  the same chat session, not derivable from model output (ADR-0013).
- MCP tool server validates that the approval token's signer is on the
  operator list.
- Allowlisted repos only; planner cannot read arbitrary URLs.

### T2 — Implementer is prompt-injected to open a malicious PR

**Vector:** repo contents, issue bodies, or test output contain instructions
that try to make the implementer push secrets, disable validate, or open a
PR with backdoored code.

**Mitigations:**
- PR opening is hook-gated, not prompt-gated (ADR-0003). Model output
  cannot relax the gate.
- `validate` is opaque (ADR-0004). The agent cannot learn what to bypass.
- Every PR is human-reviewed (design invariant; no auto-merge).
- Tool allowlist blocks outbound curl/wget; egress NetworkPolicy blocks
  non-allowlisted hosts even if a tool slips through.
- Per-run credentials expire with the pod.

### T3 — Implementer tries to exfiltrate secrets

**Vector:** agent reads GitHub token from env or file and encodes it into a
commit, PR body, or network request.

**Mitigations:**
- Token is mounted as a file, not env var; file is readable only by the
  `git` uid, not the `claude` uid.
- NetworkPolicy restricts egress to: GitHub API, CI API, validate service,
  audit sink. No generic internet.
- Pre-commit hook scans for token patterns and refuses the commit.
- CI egress is from GitHub Actions runners, not the pod.
- Installation token is scoped to one repo, `contents` + `pull_requests`
  only; cannot read orgs or other repos.

### T4 — Implementer infinite-loops or burns tokens

**Vector:** agent keeps failing validate and retrying indefinitely.

**Mitigations:**
- Loop budgets enforced by hooks (ADR-0009): wall-clock, tool-call count,
  validate-call count, retry count, token count.
- Budget breach → pod exits with a structured failure report.

### T5 — Tool allowlist silently expanded

**Vector:** attacker (internal or compromised account) adds a dangerous
command to the allowlist to enable exfiltration on a future run.

**Mitigations:**
- `config/implementer-tools.yaml` is CODEOWNERS-protected requiring two
  approvals from a short list of security-team reviewers (ADR-0005).
- Branch protection on `main` enforces the two-review rule.
- Audit log records every implementer run's effective allowlist hash;
  divergence from main is an alertable event.

### T6 — Validate tool compromised or bypassed

**Vector:** the validate binary in the pod is replaced, or the pod reads
what validate checks and crafts output that passes without really passing.

**Mitigations:**
- Validate runs as a separate service on a separate pod, not inside the
  implementer pod. Implementer calls it over a unix socket or localhost
  HTTP with a per-run token.
- Validate service image is pinned by digest.
- Validate's implementation is not shipped to the implementer pod (ADR-0004).
- Hook records the validate result server-side in the audit log; the
  implementer cannot forge a "pass" because the hook checks the audit
  record, not the agent's claim.

### T7 — CI check bypass

**Vector:** agent opens a PR claiming CI passed when it did not.

**Mitigations:**
- Hook calls GitHub API directly to verify the check run status on the
  pushed commit before allowing `pr create`.
- Required status checks configured on the target branch.

### T8 — Human approval token replay

**Vector:** an old signed approval is replayed to dispatch a different run.

**Mitigations:**
- Approval token binds to a content hash of the hydrated prompt.
- Tokens are single-use; MCP server tracks redeemed tokens.
- Tokens expire 10 minutes after issuance.

### T9 — Chat adapter compromise

**Vector:** an attacker takes over the Discord bot token and sends messages
as the bot.

**Mitigations:**
- Chat adapter cannot dispatch — it only relays messages to the planner.
- Human approval still required via user-authored chat message, not bot
  message.
- Bot tokens stored in AWS Secrets Manager, rotated quarterly.

### T10 — Supply-chain attack on implementer base image

**Vector:** the Claude Code container image pulls a malicious dependency.

**Mitigations:**
- Pin the Claude Code image by digest.
- Build our own base image from pinned upstream layers.
- No `npm install` / `pip install` at pod runtime — dependencies baked
  into the image.
- ECR scanning on push.

## Residual risks

- A PR reviewer who rubber-stamps a malicious PR is a real failure mode.
  Mitigation is process, not technical: require two reviewers for changes
  to security-sensitive paths (see CODEOWNERS in ADR-0005).
- Prompt injection in test output during validate runs could still confuse
  the agent's retry loop. Mitigation: validate output is structured JSON
  the agent cannot append to mid-stream.
