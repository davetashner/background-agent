# ADR-0015: IAM role design across AWS accounts

**Status:** Accepted
**Date:** 2026-04-23

## Context

Three AWS accounts exist (`bg-agent-security`, `bg-agent-dev-ctrl`,
`bg-agent-dev-impl`) plus the management account. The default access
path — `OrganizationAccountAccessRole` assumed from the management
account — grants full admin and is fine for bootstrapping but is not
appropriate for routine work:

- No separation between "I'm deploying infra" and "I'm reading audit logs."
- Too much standing privilege; one compromised session is a full takeover.
- No clean trust path for CI or GitHub Actions.
- Cannot scope a per-service workload role with principle of least
  privilege.

We also want every service to assume identity via **OIDC federation**,
not long-lived access keys or secrets stored in Kubernetes. This aligns
with the EKS IRSA pattern used in phase 6.

## Decision

### Named roles per workload account

Each of `bg-agent-security`, `bg-agent-dev-ctrl`, `bg-agent-dev-impl`
gets the following named roles. All have an explicit
`PermissionsBoundary` policy capping max privilege even if a policy
is mis-attached.

| Role | Purpose | Trust |
|---|---|---|
| `Admin` | Break-glass only. Rare usage, logged, alarms on assume. | AWS SSO permission set; no IAM users. |
| `DeployOps` | Run Terraform for this account's infra. | AWS SSO + GitHub Actions OIDC (repo-gated). |
| `ReadOnly` | Read-only inspection by humans during incident response. | AWS SSO. |
| `AuditRead` | Read audit log S3 (security account only; no-op role in other accounts). | AWS SSO + investigators' SSO group. |
| `AuditWrite` | Write audit log events (workload accounts only). | IRSA for shipper/dispatcher ServiceAccounts. |
| `ServiceRole-<service>` | Per-service runtime identity in EKS. | IRSA; trust EKS OIDC provider with `sub` bound to `system:serviceaccount:<ns>:<sa>`. |

### Trust policy structure

All human-facing roles use AWS SSO; we do not provision IAM users.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<acct>:saml-provider/AWSSSO" },
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": { "StringEquals": { "SAML:aud": "https://signin.aws.amazon.com/saml" } }
  }]
}
```

Service roles use the EKS OIDC provider:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<acct>:oidc-provider/<cluster-oidc>" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "<cluster-oidc>:sub": "system:serviceaccount:<namespace>:<service-account>",
        "<cluster-oidc>:aud": "sts.amazonaws.com"
      }
    }
  }]
}
```

### PermissionsBoundary

Every role attaches a boundary policy that denies:

- `iam:CreateUser`, `iam:CreateAccessKey`
- Any action outside approved regions (`us-east-1`, `us-west-2`)
- Cross-account actions on accounts outside the org
- `kms:Decrypt` on non-owned keys

The boundary is versioned in Terraform and change-reviewed with
CODEOWNERS.

### Cross-account patterns

- Dispatcher in `dev-ctrl` assumes `ServiceRole-Dispatcher-Impl` in
  `dev-impl`. Trust policy restricts to that specific ServiceAccount in
  `dev-ctrl`'s EKS OIDC provider.
- Audit shippers in `dev-ctrl` and `dev-impl` assume
  `AuditWrite` in `bg-agent-security`. Trust policy pins both the
  source account and the exact shipper ServiceAccount subject.

### Bootstrapping

At account creation, only `OrganizationAccountAccessRole` exists.
Bootstrap is a one-time `terraform apply` from a workstation using SSO
into `OrganizationAccountAccessRole` to create the full role set; after
that, SSO users access each account through the named roles only and
`OrganizationAccountAccessRole` is reserved for emergency recovery.

## Consequences

- No long-lived credentials anywhere in the system.
- Audit log of assume-role events shows who did what.
- Every service has a narrow identity; a compromised service yields
  only its specific permissions.
- SSO setup is a prerequisite; not trivial, but necessary.
- GitHub Actions OIDC trust is scoped per workflow ref, preventing a
  compromised branch from assuming production roles.

## Alternatives considered

**Single Admin role per account assumed from management.** Simpler but
defeats separation of duties.

**IAM users with access keys.** Rejected — long-lived credentials, no
MFA enforcement path, operational nightmare for rotation.

**AWS SSO only (no IRSA).** Can't identify in-cluster workloads;
requires secret-mounted credentials.

## Related

- ADR-0005 (tool allowlist): similar CODEOWNERS-protected pattern.
- ADR-0007 (EKS topology): the account structure these roles live in.
- ADR-0014 (audit logging): `AuditWrite` and `AuditRead` roles feed this.
