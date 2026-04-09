# STDB SSM Runbooks

This folder contains the initial `SSM Automation` runbooks for the STDB organization bootstrap phase.

## Scope

These runbooks are designed for the first bootstrap phase only:

- discover the AWS Organizations root
- read the STDB org configuration from `SSM Parameter Store`
- create the required OUs
- create the required long-lived accounts
- wait for account creation to complete
- move accounts into the correct OU
- create a bootstrap role in each account
- enable opt-in Regions for the created accounts when needed

## Assumptions

These runbooks assume the following prerequisites are already satisfied:

- the `STDB` AWS Organization already exists
- execution happens from the management account or a principal with equivalent `organizations` permissions
- the config parameter exists in `SSM Parameter Store`
- the management account id is included in the config
- the standard organization access role name is included in the config
- if the config parameter is a `SecureString` encrypted with a customer-managed KMS key, the automation role also has `kms:Decrypt` on that key
- the SSM automation assume role has permissions for:
  - `ssm:GetParameter`
  - `kms:Decrypt` when `WithDecryption=True` is used with a customer-managed KMS key
  - `organizations:ListRoots`
  - `organizations:ListOrganizationalUnitsForParent`
  - `organizations:CreateOrganizationalUnit`
  - `organizations:ListAccounts`
  - `organizations:CreateAccount`
  - `organizations:DescribeCreateAccountStatus`
  - `organizations:ListParents`
  - `organizations:MoveAccount`
  - `sts:AssumeRole`
  - `iam:GetRole`
  - `iam:CreateRole`
  - `iam:UpdateAssumeRolePolicy`
  - `iam:AttachRolePolicy`
  - `iam:PassRole` (for the automation role itself, so the parent orchestrator can pass it to child runbooks)
  - `ssm:StartAutomationExecution`
  - `ssm:GetAutomationExecution`
  - `ssm:DescribeAutomationExecutions`
  - `ssm:DescribeAutomationStepExecutions`
  - `ssm:DescribeDocument`
  - `ssm:GetDocument`

## Account Email Model

AWS requires a globally unique email address for every account. This is a hard AWS constraint that applies regardless of how the organization is managed.

The STDB design uses a **centralised root model** where all account root emails route to a single shared mailbox using plus-addressing aliases:

```
aws+stdb-log-archive@yourdomain.com
aws+stdb-delegated-admin@yourdomain.com
aws+stdb-network@yourdomain.com
aws+stdb-processing@yourdomain.com
aws+stdb-recovery@yourdomain.com
aws+stdb-vault@yourdomain.com
```

All aliases deliver to the same `aws@yourdomain.com` inbox. AWS treats each alias as a distinct email, satisfying the uniqueness requirement while keeping root access centralised.

### Requirements

- the base address must be a **shared or group mailbox**, not a personal account
- the mail provider must support plus-addressing (Gmail, Google Workspace, Microsoft 365, and most providers do)
- the mailbox must be monitored — AWS sends account verification and root password reset emails to these addresses
- MFA for account root should be configured using a centralised mechanism (hardware tokens or a shared authenticator vault)

### Why This Matters

- root password resets for any account go to one controlled inbox
- no dependency on individual team members for account root access
- account root credentials remain under centralised governance
- the mailbox survives team changes and role rotations

## Parameter Store Config

The master org config is expected at a parameter such as:

- `/stdb/org/bootstrap/config`

It should follow the lean v1 schema documented in:

- [STDB-org-bootstrap-approach.md](/Users/krishnansubramanian/Documents/New%20project/STDB-org-bootstrap-approach.md)

## Runbooks

- `Bootstrap-STDB-Organization.yaml`
- `Discover-Root.yaml`
- `Ensure-OUs.yaml`
- `Create-Accounts.yaml`
- `Wait-For-Account-Creation.yaml`
- `Move-Accounts-To-OUs.yaml`
- `Create-Bootstrap-Role.yaml`
- `Enable-Account-Regions.yaml`

## Notes

- The child runbooks are designed to be independently re-runnable.
- The orchestrator keeps the first flow simple by sequencing the child runbooks.
- The bootstrap role runbook currently attaches `AdministratorAccess` to keep the first phase practical. This should be tightened in a later phase.
- The Region enablement runbook is kept separate from the main bootstrap orchestrator so opt-in Region activation can be staged before SCP-based Region restrictions are introduced.
