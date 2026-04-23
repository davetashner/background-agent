# ADR-0006: Per-run ephemeral Kubernetes namespaces

**Status:** Accepted
**Date:** 2026-04-22

## Context

Each implementer run handles hostile input (repo code, test output, LLM
output). Sharing namespaces across runs would mean one poisoned run could
plant persistence that affects the next. Sharing pods is worse.

## Decision

- One Kubernetes `Namespace` per run, named `impl-<run-id>`.
- One `Job` per namespace, backed by a single Pod.
- Namespace, ServiceAccount, NetworkPolicy, and Secret are all created
  by the dispatcher at run start and deleted on run end (including
  failures).
- A finalizer ensures cleanup even if the dispatcher crashes mid-run.
- The implementer pod runs on a dedicated node group with:
  - `gVisor` runtime class for syscall isolation.
  - Taints preventing non-implementer workloads from landing there.
  - Restricted PodSecurityStandard.
- Default-deny NetworkPolicy; egress explicitly allowed only to:
  - `api.github.com` (for git and PR operations).
  - The validate service's ClusterIP.
  - The audit log sink.

## Consequences

- State between runs is physically impossible to share. No cache, no
  warm pod pool. First-run latency is higher; acceptable for a
  human-in-the-loop tool.
- gVisor has a performance hit (~10-20% for syscall-heavy workloads).
  Acceptable.
- Orphaned namespaces on dispatcher crashes are handled by the
  finalizer plus a reconciler CronJob that sweeps stale `impl-*`
  namespaces after a timeout.

## Alternatives considered

**Namespace per user, pod per run.** Rejected — shared namespace means
shared ConfigMaps, Secrets, and Service accounts; one compromised pod
can observe the next.

**Pod-per-run in a shared namespace.** Rejected — NetworkPolicies and
RBAC are namespace-scoped; isolation is weaker.

**Firecracker microVMs.** Better isolation but much more operational
burden. Revisit if gVisor proves inadequate.
