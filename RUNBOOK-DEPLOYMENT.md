# STDB Runbook Deployment Guide

This guide describes how to set up and run the initial STDB organization bootstrap runbooks.

## Purpose

The runbooks in this folder are intended for the first bootstrap phase only:

- create the required OUs
- create the required long-lived accounts
- move accounts into the correct OUs
- create a bootstrap role in each account
- enable opt-in Regions where required

## Prerequisites

Before running these automations, the following should already be true:

- the `STDB` AWS Organization already exists
- you are operating from the management account for that organization
- `SSM Automation` is available in the target region
- the automation execution role exists and has the required permissions
- the org bootstrap config has been stored in `SSM Parameter Store`
- if the config parameter is a `SecureString` using a customer-managed KMS key, the automation role also has `kms:Decrypt` on that key
- CloudWatch Logs is configured for SSM Automation executeScript output (see the CLI commands guide for setup)
- AWS Organizations trusted access for IAM is required only when running the standalone `Secure-Root-Access` runbook. That runbook enables it automatically.

## Files

Runbooks:

- [Bootstrap-STDB-Organization.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Bootstrap-STDB-Organization.yaml)
- [Validate-Config.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Validate-Config.yaml)
- [Discover-Root.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Discover-Root.yaml)
- [Ensure-OUs.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Ensure-OUs.yaml)
- [Create-Accounts.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Create-Accounts.yaml)
- [Wait-For-Account-Creation.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Wait-For-Account-Creation.yaml)
- [Move-Accounts-To-OUs.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Move-Accounts-To-OUs.yaml)
- [Create-Bootstrap-Role.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Create-Bootstrap-Role.yaml)
- [Secure-Root-Access.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Secure-Root-Access.yaml)
- [Enable-Account-Regions.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Enable-Account-Regions.yaml)
- [Verify-Bootstrap.yaml](/Users/krishnansubramanian/Documents/New%20project/ssm/Verify-Bootstrap.yaml)

Sample config:

- [examples/stdb-org-bootstrap-config.json](/Users/krishnansubramanian/Documents/New%20project/ssm/examples/stdb-org-bootstrap-config.json)

## Account Email Model

AWS requires a globally unique email address for every account. There is no way to create an account without one, even when using a centralised root management model.

The recommended approach for STDB is to use **plus-addressing aliases** on a single shared mailbox:

```
aws+stdb-log-archive@yourdomain.com
aws+stdb-delegated-admin@yourdomain.com
aws+stdb-key-vault@yourdomain.com
aws+stdb-network@yourdomain.com
aws+stdb-processing@yourdomain.com
aws+stdb-recovery@yourdomain.com
aws+stdb-vault@yourdomain.com
```

All of these deliver to the same `aws@yourdomain.com` inbox. AWS treats each alias as a unique email address.

### Centralised Root Requirements

- use a **shared or group mailbox** as the base address — not a personal email
- the mail provider must support plus-addressing (Gmail, Google Workspace, Microsoft 365, and most providers do)
- the mailbox must be actively monitored — AWS sends verification emails, root password resets, and account notifications to these addresses
- configure MFA for each account's root user using a centralised mechanism (hardware tokens or a shared authenticator vault)
- document which alias maps to which account and store this alongside the org config

### Why Centralised Root

- one team controls root access to all accounts
- root password resets are not dependent on any individual
- the mailbox survives team changes and role rotations
- aligns with the STDB principle of centralised governance over the organization

### Before You Begin

Confirm the following for your email setup:

- the base mailbox exists and is accessible to the platform team
- plus-addressing is confirmed working (send a test to `aws+test@yourdomain.com`)
- each alias you plan to use in the config is not already associated with an existing AWS account
- the mailbox has appropriate access controls and is not publicly reachable

## Step 1: Store the Config in Parameter Store

Use the sample config as the starting point and adjust:

- `managementAccountId`
- `roleName`
- `orgAccessRoleName`
- OU names if needed
- account names and emails (see the Account Email Model section above)

Recommended parameter name:

- `/stdb/org/bootstrap/config`

The runbooks expect the config to be stored as JSON in a single parameter for the first version.

## Step 2: Create the Automation Execution Role

Create the IAM role that `SSM Automation` will assume when running the documents.

This is the value that will be passed as:

- `AutomationAssumeRole`

At minimum, that role should have permissions for:

- `ssm:GetParameter`
- `kms:Decrypt` when the config is read with `WithDecryption=True` and stored under a customer-managed KMS key
- `organizations:ListRoots`
- `organizations:ListOrganizationalUnitsForParent`
- `organizations:CreateOrganizationalUnit`
- `organizations:ListAccounts`
- `organizations:CreateAccount`
- `organizations:DescribeCreateAccountStatus`
- `organizations:ListParents`
- `organizations:MoveAccount`
- `account:ListRegions`
- `account:EnableRegion`
- `sts:AssumeRole`
- `iam:GetRole`
- `iam:CreateRole`
- `iam:UpdateAssumeRolePolicy`
- `iam:AttachRolePolicy`
- `iam:PassRole` (for the automation role itself — required so the parent orchestrator can pass the role to child runbook executions)
- `ssm:StartAutomationExecution`
- `ssm:GetAutomationExecution`
- `ssm:DescribeAutomationExecutions`
- `ssm:DescribeAutomationStepExecutions`
- `ssm:DescribeDocument`
- `ssm:GetDocument`
- `logs:CreateLogGroup`
- `logs:CreateLogStream`
- `logs:DescribeLogGroups`
- `logs:DescribeLogStreams`
- `logs:PutLogEvents`
- `organizations:EnableAWSServiceAccess`
- `organizations:ListAWSServiceAccessForOrganization`
- `iam:EnableOrganizationsRootCredentialsManagement`
- `iam:EnableOrganizationsRootSessions`
- `iam:ListOrganizationsFeatures`
- `sts:AssumeRoot`

## Step 3: Create the SSM Automation Documents

Upload each YAML file in this folder as a separate `SSM Automation` document.

Recommended document names:

- `Bootstrap-STDB-Organization`
- `Validate-Config`
- `Discover-Root`
- `Ensure-OUs`
- `Create-Accounts`
- `Wait-For-Account-Creation`
- `Move-Accounts-To-OUs`
- `Create-Bootstrap-Role`
- `Secure-Root-Access`
- `Enable-Account-Regions`
- `Verify-Bootstrap`

The parent runbook references the child runbooks by these names unless you override them through parameters.

## Step 4: Validate the Config Before First Run

Before running the automation, verify:

- the `managementAccountId` is correct
- all account email addresses are unique, valid, and deliverable
- each email alias resolves to the centralised shared mailbox
- none of the email addresses are already associated with an existing AWS account
- the `orgAccessRoleName` matches the role name created during `CreateAccount`
- the config parameter path matches the runbook input

You can also run `Validate-Config` as a standalone check before attempting the full bootstrap. It will verify the config schema, field uniqueness, and OU key references without making any changes to the organization.

## Recommended First Execution Strategy

For the first rollout, do not start with the full parent orchestration immediately.

Use a staged execution so you can verify each layer before continuing.

### Phase A: Validate Organization Discovery and OU Creation

Run in this order:

1. `Discover-Root`
2. `Ensure-OUs`

Verify:

- the organization root is discovered correctly
- the expected OUs exist under the root

### Phase B: Create the Accounts

Run in this order:

1. `Create-Accounts`
2. `Wait-For-Account-Creation`

Verify:

- the expected accounts are created
- account creation completes successfully
- the default organization access role is present in each new account

### Phase C: Place the Accounts

Run:

1. `Move-Accounts-To-OUs`

Verify:

- each account is placed into the intended OU

### Phase D: Create the Bootstrap Role

Run:

1. `Create-Bootstrap-Role`

Verify:

- the bootstrap role exists in each account
- the management account can assume it

### Phase E: Enable Opt-In Regions

Run:

1. `Enable-Account-Regions`

Example for the current plan:

- `RegionNames = ap-south-2`

Verify:

- the runbook requested enablement for `ap-south-2` where needed
- accounts already in `ENABLED`, `ENABLED_BY_DEFAULT`, or `ENABLING` state were skipped or recorded without duplicate requests
- `ap-south-2` later reaches `ENABLED` state in the target accounts
- no unexpected Regions were requested by the runbook

## Full Execution Order

Once the first staged rollout is successful, the normal sequence is:

1. `Validate-Config`
2. `Discover-Root`
3. `Ensure-OUs`
4. `Create-Accounts`
5. `Wait-For-Account-Creation`
6. `Move-Accounts-To-OUs`
7. `Create-Bootstrap-Role`
8. `Enable-Account-Regions`
9. `Verify-Bootstrap`

Or simply run:

1. `Bootstrap-STDB-Organization` (steps 1-7 are automated by the parent orchestrator)
2. `Enable-Account-Regions` (separate, staged after bootstrap)
3. `Verify-Bootstrap` (post-run verification)

`Secure-Root-Access` is intentionally not part of the parent orchestrator. Run it as a separate post-bootstrap hardening step only after account bootstrap has been verified.

All executions emit structured JSON logs with a RunId correlation ID. When CloudWatch Logs is enabled for SSM Automation, use CloudWatch Logs Insights to trace a full bootstrap run across all child execution log streams.

## Inputs to Use

### Parent Runbook

Run `Bootstrap-STDB-Organization` with:

- `AutomationAssumeRole`
- `ConfigParameterName`

Example:

- `AutomationAssumeRole = arn:aws:iam::<management-account-id>:role/<automation-role>`
- `ConfigParameterName = /stdb/org/bootstrap/config`
- `RunId` (optional — auto-generated if omitted)

### Child Runbooks

Each child runbook takes:

- `AutomationAssumeRole`
- `ConfigParameterName` for all runbooks except `Discover-Root`

`Discover-Root` takes only:

- `AutomationAssumeRole`

`Enable-Account-Regions` also takes:

- `RegionNames`
- optional `AccountEmails`

`Secure-Root-Access` also takes:

- `AutomationAssumeRole` (required)
- `ConfigParameterName` (required)
- `RunId` (optional — auto-generated if omitted)

## Re-run Behavior

These runbooks are intended to be re-runnable.

Expected behavior on re-run:

- existing OUs are skipped
- existing accounts are skipped
- accounts already in the correct OU are not moved
- bootstrap roles that already exist are updated or left in place

The main caution remains account creation. Each account email must be globally unique across all of AWS. When using the centralised root email model with plus-addressing, ensure no alias is reused or already claimed by another AWS account.

## Suggested First-Day Checklist

Before execution:

- shared mailbox exists and plus-addressing is confirmed working
- each account email alias is unique and not tied to an existing AWS account
- config stored in Parameter Store
- automation role created
- all documents uploaded
- account emails reviewed against the centralised mailbox
- management account id confirmed

After execution:

- OUs verified
- accounts verified
- account placement verified
- bootstrap role verified
- opt-in Regions verified where applicable
- `Verify-Bootstrap` run confirms overall state matches config (or run manually to check specific mismatches)

## Next Phase

Once this bootstrap phase is stable, the next phase can use the bootstrap role to begin the STDB landing zone implementation.
