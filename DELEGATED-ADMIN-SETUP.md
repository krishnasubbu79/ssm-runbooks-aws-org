# STDB Delegated Admin Setup

> **Note:** Decision DL-006 ("Delegated admin security service model") is currently in **Proposed** status. The service list below may change once DL-006 is finalized.

This document provides the AWS CLI commands and minimum IAM permissions required to register the `stdb-delegated-admin` account as the delegated administrator for centralized security and governance services across the STDB AWS Organization.

Replace the placeholder values before running:

- `<DELEGATED_ADMIN_ACCOUNT_ID>` — the account ID for `stdb-delegated-admin`
- `<REGION>` — the target AWS region (e.g., `ap-south-1`)

All commands use `--profile orga`. Ensure this profile is configured in your AWS CLI credentials and points to the management account.

---

## Prerequisites

- STDB organization bootstrap is complete (OUs and accounts exist)
- `stdb-delegated-admin` account is created and placed in the Security OU
- Caller has credentials that can assume `STDBBootstrapRole` in the management account
- AWS Organizations has **All features** enabled (not consolidated billing only)

---

## Service Principal Reference

| Service | Service Principal |
|---|---|
| GuardDuty | `guardduty.amazonaws.com` |
| Security Hub | `securityhub.amazonaws.com` |
| AWS Config | `config.amazonaws.com` |

---

## Step 1 — Enable Trusted Access

Enable AWS Organizations trusted access for each service. This allows the service to operate across the organization.

### GuardDuty

```bash
aws organizations enable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "guardduty.amazonaws.com"
```

### Security Hub

```bash
aws organizations enable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "securityhub.amazonaws.com"
```

### AWS Config

```bash
aws organizations enable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "config.amazonaws.com"
```

### Verify trusted access

```bash
aws organizations list-aws-service-access-for-organization \
  --profile orga \
  --region <REGION> \
  --query "EnabledServicePrincipals[].ServicePrincipal" \
  --output table
```

All three service principals should appear in the output.

---

## Step 2 — Register Delegated Administrator

Register the `stdb-delegated-admin` account as the delegated administrator for each service.

### GuardDuty

```bash
aws organizations register-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "guardduty.amazonaws.com"
```

### Security Hub

```bash
aws organizations register-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "securityhub.amazonaws.com"
```

### AWS Config

```bash
aws organizations register-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "config.amazonaws.com"
```

### Verify delegation

List all delegated administrators:

```bash
aws organizations list-delegated-administrators \
  --profile orga \
  --region <REGION> \
  --output table
```

List all delegated services for the account:

```bash
aws organizations list-delegated-services-for-account \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --output table
```

All three service principals should appear under the `stdb-delegated-admin` account.

---

## Step 3 — Verify from the Delegated Admin Account

Run these commands using credentials in the `stdb-delegated-admin` account to confirm each service recognises it as the delegated admin.

### GuardDuty

```bash
aws guardduty list-organization-admin-accounts \
  --region <REGION>
```

The output should list `<DELEGATED_ADMIN_ACCOUNT_ID>` with status `ENABLED`.

### Security Hub

```bash
aws securityhub list-organization-admin-accounts \
  --region <REGION>
```

The output should list `<DELEGATED_ADMIN_ACCOUNT_ID>` with status `ENABLED`.

### AWS Config

```bash
aws organizations list-delegated-administrators \
  --service-principal "config.amazonaws.com" \
  --region <REGION>
```

The output should list `<DELEGATED_ADMIN_ACCOUNT_ID>`.

---

## Minimum IAM Policy for Delegation

The following IAM policy is the minimum required on the management-account role performing the delegation. `STDBBootstrapRole` currently has `AdministratorAccess` which covers these permissions. This policy documents the scoped-down set for when the bootstrap role is replaced (tracked in DL-010).

Attach this policy to the role in the **management account** that will execute the commands above.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableTrustedAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:EnableAWSServiceAccess",
        "organizations:DisableAWSServiceAccess",
        "organizations:ListAWSServiceAccessForOrganization"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DelegatedAdminManagement",
      "Effect": "Allow",
      "Action": [
        "organizations:RegisterDelegatedAdministrator",
        "organizations:DeregisterDelegatedAdministrator",
        "organizations:ListDelegatedAdministrators",
        "organizations:ListDelegatedServicesForAccount"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OrganizationReadAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:DescribeOrganization",
        "organizations:ListAccounts",
        "organizations:DescribeAccount"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Note:** AWS Organizations APIs do not support resource-level permissions. `Resource: "*"` is required for all Organizations actions.

---

## Rollback — Deregister Delegated Administrator

To remove delegation for a service, deregister the account first, then optionally disable trusted access.

### Deregister

```bash
aws organizations deregister-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "guardduty.amazonaws.com"
```

```bash
aws organizations deregister-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "securityhub.amazonaws.com"
```

```bash
aws organizations deregister-delegated-administrator \
  --profile orga \
  --region <REGION> \
  --account-id <DELEGATED_ADMIN_ACCOUNT_ID> \
  --service-principal "config.amazonaws.com"
```

### Disable trusted access (optional)

> **Warning:** Disabling trusted access may disrupt existing service configurations across all member accounts. Only disable trusted access if you are certain the service is no longer needed at the organization level. Always deregister the delegated admin before disabling trusted access.

```bash
aws organizations disable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "guardduty.amazonaws.com"
```

```bash
aws organizations disable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "securityhub.amazonaws.com"
```

```bash
aws organizations disable-aws-service-access \
  --profile orga \
  --region <REGION> \
  --service-principal "config.amazonaws.com"
```

---

## Security Notes

1. **All delegation commands must run from the management account.** Only the management account can enable trusted access and register delegated administrators.
2. **One delegated admin per service.** Most services allow only one delegated administrator account. Changing it requires deregistering the current one first.
3. **The management account cannot be a delegated admin.** AWS does not allow registering the management account itself as a delegated administrator.
4. **Service-specific onboarding is separate.** Registering a delegated admin is the Organizations-level step only. Each service requires additional setup (e.g., enabling GuardDuty detectors, enabling Security Hub standards, creating Config aggregators) from within the delegated admin account. That configuration is out of scope for this document.
5. **DL-006 is Proposed.** The service list (GuardDuty, Security Hub, AWS Config) is subject to change. Update this document when DL-006 is finalized.
