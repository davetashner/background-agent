# ADR-0014: Append-only audit log in S3 Object Lock

**Status:** Accepted
**Date:** 2026-04-22

## Context

The user explicitly requires that the system prompt from planner to
implementer always be logged for later audit. In practice we want more
than that: every action that influenced a code change should be
reconstructible months later.

## Decision

- Single logical audit log, append-only, with S3 Object Lock
  (Compliance mode) preventing deletion for the retention period.
- Bucket lives in the `bg-agent-security` account; workload accounts
  have write-only access via cross-account role (no list, no get,
  no delete).
- Every event is a single JSON object keyed by `event_id`, sharded into
  prefixes by date and run ID.

## Event categories

| Category | When |
|---|---|
| `chat.message` | Every message to/from a chat adapter. |
| `planner.tool_call` | Every MCP tool call the planner makes. |
| `approval.issued` | Each human-approval token, with identity. |
| `dispatch.requested` | Dispatch parameters including prompt hash. |
| `dispatch.launched` | Pod launched; full system prompt & user prompt. |
| `implementer.tool_call` | Every tool call the implementer makes. |
| `implementer.validate` | Every validate invocation + result. |
| `implementer.ci_check` | Each CI poll result. |
| `implementer.pr_opened` | PR URL, commit SHA, reviewer assignment. |
| `implementer.outcome` | Final outcome + reason. |
| `allowlist.drift` | Deployed allowlist hash vs. main. |

## Guarantees

- **Integrity:** events are written via a dedicated shipper service
  that signs each event. Compliance-mode Object Lock prevents deletion
  until retention expires.
- **Availability:** writes are retried with a local buffer; a
  persistent write failure blocks the run (no silent logging gaps
  during a production run).
- **Privacy:** audit entries may contain source code and proprietary
  plans. Bucket access is tightly scoped; entries are encrypted with
  KMS keys in the security account.
- **Retention:** 7 years default (revisit in a later ADR).

## Consequences

- Storage cost is bounded; expected volume is small relative to S3.
- Debugging requires audit log access for anyone investigating a run.
  That's a feature, not a bug — investigation is authorized.
- Object Lock means mistakes (accidental log of a secret) are hard to
  undo. We mitigate by scrubbing credentials at the shipper layer
  before write.

## Alternatives considered

**CloudWatch Logs.** Mutable enough to be problematic for forensics
under insider threat; rejected for the authoritative log. CloudWatch
is fine for operational logs (which events did the dispatcher see),
but not the audit log of record.

**Database (DynamoDB / RDS).** Harder to make truly append-only;
recovery story more complex.

**Self-hosted WORM storage.** More operational burden than S3 Object
Lock for the same guarantee.
