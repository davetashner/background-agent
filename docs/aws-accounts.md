# AWS accounts

Live record of AWS account IDs for this project. Update when accounts are
created, renamed, or closed.

See ADR-0007 for the topology rationale.

## Management / root

| Account | ID | Purpose |
|---|---|---|
| `aws-management` | `724554527674` | AWS Organizations management account (`davetashner+aws-management@gmail.com`). Holds the org and assumes into member accounts via `OrganizationAccountAccessRole`. |

## Member accounts — current

| Account | ID | Environment | Role |
|---|---|---|---|
| `bg-agent-security` | `204758922821` | Cross-env | Audit log archive (S3 Object Lock), CloudTrail org trail sink, GuardDuty admin, VPC Flow Logs sink. |
| `bg-agent-dev-ctrl` | `966579633789` | dev | Control-plane services: planner, MCP server, dispatcher, chat adapters, audit shipper. |
| `bg-agent-dev-impl` | `691537866817` | dev | Implementer pods only. Cross-account dispatch target from `bg-agent-dev-ctrl`. |

## Member accounts — planned (future phases)

| Account | Purpose | Status |
|---|---|---|
| `bg-agent-stg-ctrl` | Staging control plane | Not created |
| `bg-agent-stg-impl` | Staging implementer pods | Not created |
| `bg-agent-prd-ctrl` | Prod control plane | Not created |
| `bg-agent-prd-impl` | Prod implementer pods | Not created |

## Access

Each member account has the default `OrganizationAccountAccessRole` with
a trust policy allowing the management account (`724554527674`). From a
session in the management account:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<member-account-id>:role/OrganizationAccountAccessRole \
  --role-session-name bootstrap
```

A named role per member account scoped to day-to-day ops work is planned
as a follow-up — see the IAM role design issue in the backlog.

## What still needs doing before these accounts are "usable"

Tracked in beads; summary here:

- Baseline: CloudTrail org trail to security account, GuardDuty org-wide,
  Config aggregator, SCPs denying root-user actions and non-approved
  regions.
- IAM role design: named roles per account for specific duties (ops,
  deploy, audit-read, audit-write) with least privilege. No humans in
  `OrganizationAccountAccessRole` for routine work.
- Budget alarms per account.
- Service quotas review (EKS clusters, NAT gateways, etc.) before phase 6.
