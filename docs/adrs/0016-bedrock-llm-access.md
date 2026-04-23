# ADR-0016: Bedrock for LLM access

**Status:** Accepted
**Date:** 2026-04-23

## Context

The prototype needs LLM capacity for the planner and for the
implementer (Claude Code running in a pod). Two main options:

- **Anthropic API direct** — fastest to stand up, requires distributing
  long-lived API keys to each service (or a secret broker).
- **AWS Bedrock** — IAM-based auth (no API keys), API traffic stays
  within the AWS account, prompt caching is supported (different
  semantics from Anthropic API), Claude models are available.

All other prototype infrastructure runs in AWS. Keeping LLM access in
AWS aligns auth with the rest of the system.

## Decision

Use **AWS Bedrock** for all Claude LLM calls in this prototype.
Services use the default AWS credential chain — IRSA in EKS, SSO for
workstations, GitHub Actions OIDC for CI.

### Model selection

All services target Claude models via Bedrock model IDs. The
application-layer abstraction maps a logical model name to the Bedrock
ID and region, so we can switch regions or models without touching
application code.

### Regions

- Primary: `us-east-1` for broadest model availability.
- Secondary: `us-west-2` for failover.
- Both regions explicitly enabled in `bg-agent-dev-ctrl` and
  `bg-agent-dev-impl` via `bedrock:InvokeModel` on approved model IDs.

### IAM

New IAM policy `BedrockInvokeClaude` attached to the ServiceRole of
each service that needs LLM access:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    "Resource": [
      "arn:aws:bedrock:us-east-1::foundation-model/anthropic.*",
      "arn:aws:bedrock:us-west-2::foundation-model/anthropic.*"
    ]
  }]
}
```

The planner's and implementer's ServiceAccount IRSA roles include this
policy.

### Claude Code inside implementer pods

Claude Code is configured (via `ANTHROPIC_BEDROCK=1` or equivalent env
flag / settings) to use the Bedrock transport. Credentials come from
the pod's IRSA role; no secret mounting. The allowlist configuration
still governs which tools and commands the Claude Code session can run.

### Prompt caching

Bedrock supports prompt caching for Claude models with slightly
different semantics from the Anthropic API (explicit `cachePoint`
markers, narrower TTL). Services must:

- Place cache breakpoints at stable boundaries (system prompt, tool
  list, static context) — verified by a unit test that refuses to add
  a breakpoint inside dynamic content.
- Log cache hit rate to audit for each run. Target: ≥60% cache hit rate
  on the implementer system prompt after warmup.

### Observability

- CloudWatch metrics per-account: `bedrock:InvokeModel` count,
  throttles, token counts (input/output/cached).
- Per-run cache hit rate in the audit log.
- Budget alarm on Bedrock spend separate from other services.

## Consequences

- No Anthropic API keys anywhere. Rotation is a non-issue.
- Auth model matches every other AWS service (IRSA everywhere).
- Model availability is gated by AWS; new Claude releases may land on
  Bedrock with a delay.
- We lock into Bedrock's prompt caching semantics; migrating to
  Anthropic API direct later would require a caching-layer rewrite.
- Per-region model enablement is an operational step for each new
  account.

## Alternatives considered

**Anthropic API direct with secret broker.** Rejected for the
prototype — adds a secret-management surface we don't otherwise need.
If Bedrock turns out to have unacceptable latency or caching issues in
phase 6, revisit.

**Azure OpenAI / Vertex / other providers.** Off-table — the user
specified AWS and Claude models.

**Multi-provider abstraction layer.** Over-engineering for a
prototype. Model access is behind a single adapter, easy to swap
later.

## Related

- ADR-0007 (AWS topology): the accounts this runs in.
- ADR-0015 (IAM role design): the roles that get the `BedrockInvokeClaude`
  policy.
- ADR-0009 (loop budgets): token budgets are measured against Bedrock
  input/output token counts.
