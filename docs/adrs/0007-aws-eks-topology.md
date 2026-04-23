# ADR-0007: AWS EKS topology

**Status:** Accepted
**Date:** 2026-04-22

## Context

User is deploying on AWS EKS. We need a topology that isolates the
implementer's blast radius without forcing us to run many clusters.

The implementer pods are the least-trusted component: they execute
model-generated code against repos with potentially hostile contents.
Keeping them in the same IAM account as the planner, dispatcher, and
audit sink would mean one gVisor escape could pivot into the control
plane.

## Decision

**AWS Organizations layout:**

```
Org root
├── OU: Security
│   └── acct: bg-agent-security       # CloudTrail archive, GuardDuty, audit log S3
├── OU: Workloads
│   ├── acct: bg-agent-dev            # everything for developer iteration
│   ├── acct: bg-agent-stg-ctrl       # staging control plane
│   ├── acct: bg-agent-stg-impl       # staging implementer pods
│   ├── acct: bg-agent-prd-ctrl       # prod control plane
│   └── acct: bg-agent-prd-impl       # prod implementer pods
```

**Per environment (stg, prd):**

- Two accounts: `*-ctrl` and `*-impl`, giving an IAM trust boundary
  between the planner/dispatcher/MCP-server and the implementer pods.
- One EKS cluster per account.
- Control-plane cluster: planner, dispatcher, MCP server, chat adapters,
  audit-shipping service.
- Implementer cluster: receives namespace/Job creation requests from the
  dispatcher in the control plane via a cross-account, audience-scoped
  token (OIDC federation, no static creds).
- Implementer cluster has no inbound API access from the internet; only
  the dispatcher's IRSA role can create namespaces.

**For dev:**

- Single account `bg-agent-dev` with one EKS cluster.
- Control plane and implementer workloads in separate namespaces, not
  separate accounts. Explicitly lower isolation for iteration speed.
- Audit log goes to the security account's S3 bucket even from dev
  (cross-account write), so we practice the prod path.

**Shared:**

- CloudTrail org trail archives to `bg-agent-security`.
- GuardDuty enabled org-wide.
- VPC Flow Logs from implementer VPCs to security account.

## Consequences

- Cross-account dispatch is more complex than same-account; worth it for
  the IAM boundary.
- Two clusters per env (stg, prd) is more ops burden than one; mitigated
  by the dev-single-cluster choice and by heavy use of managed addons.
- Cost is noticeably higher than a minimal deploy. Acceptable given
  security priority.

## Alternatives considered

**Single account, single cluster, namespace isolation only.** Rejected —
one gVisor escape pivots into the planner. The IAM boundary is the
point.

**One cluster per implementer pod.** Rejected — cluster create time and
cost are both prohibitive at any real throughput.

**EKS Fargate for implementer pods.** Attractive: per-pod kernel
isolation with no node management. Evaluate in phase 7 — may let us
drop gVisor.

**Non-AWS (GKE, Azure) alternatives.** User specified AWS.
