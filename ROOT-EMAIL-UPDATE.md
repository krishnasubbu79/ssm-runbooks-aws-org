# Root Email Update Guide

This guide explains how to change the root user email address for STDB member accounts after account creation.

AWS refers to this as the **root user email address** or **primary email address** for the account.

## Purpose

Use this guide when:

- an account was created with the wrong root email alias
- the central root mailbox naming convention changes
- an account needs to be moved to a different approved root email alias
- you need to validate or correct root email ownership after bootstrap

## Important Distinction

The root email is not the same as account contact information.

| Item | Meaning |
|---|---|
| Root email / primary email | The email used as the account root login identity |
| Primary contact | Company/contact profile information for the account |
| Alternate contacts | Billing, security, and operations contact addresses |

This guide covers only the **root email / primary email**.

## Supported Update Model

For AWS Organizations member accounts, the root email can be updated centrally from:

- the AWS Organizations management account, or
- a delegated administrator account for AWS Account Management

This avoids signing in directly as the member account root user.

For the STDB bootstrap model, this should normally be performed from the management account.

## Prerequisites

Before changing a member account root email:

- the account must be a member account in the STDB AWS Organization
- AWS Organizations must have **all features** enabled
- trusted access must be enabled for AWS Account Management
- the caller must have the required `account:*` permissions
- the new email address must be unique and not already used by another AWS account
- the platform team must be able to receive the OTP sent to the new email address

## Enable Trusted Access for Account Management

Run from the management account:

```bash
aws organizations enable-aws-service-access \
  --service-principal account.amazonaws.com \
  --profile orga
```

Verify:

```bash
aws organizations list-aws-service-access-for-organization \
  --profile orga \
  --query "EnabledServicePrincipals[?ServicePrincipal=='account.amazonaws.com']"
```

## Required IAM Permissions

The role or user performing the update from the management account needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RootEmailUpdate",
      "Effect": "Allow",
      "Action": [
        "account:GetPrimaryEmail",
        "account:StartPrimaryEmailUpdate",
        "account:AcceptPrimaryEmailUpdate"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OrganizationsRead",
      "Effect": "Allow",
      "Action": [
        "organizations:DescribeOrganization",
        "organizations:DescribeAccount",
        "organizations:ListAccounts"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EnableAccountManagementTrustedAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:EnableAWSServiceAccess",
        "organizations:ListAWSServiceAccessForOrganization"
      ],
      "Resource": "*"
    }
  ]
}
```

## Update Flow

### 1. Get the current root email

```bash
aws account get-primary-email \
  --account-id <MEMBER_ACCOUNT_ID> \
  --profile orga
```

### 2. Start the root email update

```bash
aws account start-primary-email-update \
  --account-id <MEMBER_ACCOUNT_ID> \
  --primary-email <NEW_ROOT_EMAIL> \
  --profile orga
```

AWS sends a one-time password to `<NEW_ROOT_EMAIL>`.

### 3. Retrieve the OTP

Check the mailbox for `<NEW_ROOT_EMAIL>`.

For the STDB central root mailbox model, this usually means checking the shared mailbox that receives plus-addressing aliases, for example:

```text
aws+stdb-vault@yourdomain.com
```

### 4. Accept the root email update

```bash
aws account accept-primary-email-update \
  --account-id <MEMBER_ACCOUNT_ID> \
  --primary-email <NEW_ROOT_EMAIL> \
  --otp <OTP_CODE> \
  --profile orga
```

### 5. Verify the update

```bash
aws account get-primary-email \
  --account-id <MEMBER_ACCOUNT_ID> \
  --profile orga
```

Confirm the output matches `<NEW_ROOT_EMAIL>`.

## Example

```bash
ACCOUNT_ID="123456789012"
NEW_EMAIL="aws+stdb-vault@yourdomain.com"

aws account get-primary-email \
  --account-id "$ACCOUNT_ID" \
  --profile orga

aws account start-primary-email-update \
  --account-id "$ACCOUNT_ID" \
  --primary-email "$NEW_EMAIL" \
  --profile orga

# Retrieve OTP from the NEW_EMAIL mailbox.

aws account accept-primary-email-update \
  --account-id "$ACCOUNT_ID" \
  --primary-email "$NEW_EMAIL" \
  --otp "<OTP_CODE>" \
  --profile orga

aws account get-primary-email \
  --account-id "$ACCOUNT_ID" \
  --profile orga
```

## Operational Guidance

- Prefer correcting root emails before running `Secure-Root-Access`.
- Keep the root email aliases aligned with the Parameter Store org config.
- Record every root email change in the platform change log or ticketing system.
- Do not use personal email addresses for STDB root emails.
- Confirm the new email is deliverable before starting the update.
- Keep the OTP handling process controlled because root email changes are sensitive.

## Relationship to Secure-Root-Access

`Secure-Root-Access` removes root credentials and centralizes root access management.

Changing the root email and removing root credentials are separate controls:

- root email update changes the account root login identity
- `Secure-Root-Access` removes root credentials and enables centralized root access management

Recommended sequence:

1. create account
2. verify or correct root email
3. create and verify bootstrap role
4. run `Verify-Bootstrap`
5. run `Secure-Root-Access` when ready

## Notes

- Root email updates require OTP verification through the new email address.
- The new email address must not already be associated with another AWS account.
- The management account root email is handled differently and normally requires root-console access.
- This guide is intended for member accounts in the STDB organization.
