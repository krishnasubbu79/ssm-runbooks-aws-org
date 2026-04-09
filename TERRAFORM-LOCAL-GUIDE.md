# Terraform Local Execution Guide

This guide covers how to run Terraform from your local laptop against the STDB organization using the infrastructure provisioned by the `Setup-Terraform-Local-Foundation` runbook.

## Execution Model

```
Local laptop (IAM user credentials)
  │
  └─> terraform
        │
        ├─> backend: assumes STDBBootstrapRole in mgmt account
        │     └─> reads/writes S3 state bucket (KMS encrypted)
        │
        └─> providers: assumes STDBBootstrapRole per account
              ├─> management account  → plan / apply
              ├─> security account    → plan / apply
              ├─> infra account       → plan / apply
              ├─> processing account  → plan / apply
              └─> ... all accounts identical
```

Your IAM user is used only for authentication.

Terraform operations are split into two paths:

- backend/state access uses `STDBBootstrapRole` in the management account
- provider/resource operations use `STDBBootstrapRole` in the target account

This keeps Terraform state access centralized in the management account while still allowing cross-account infrastructure creation.

The local-foundation runbook is intended to be idempotent at both the resource and policy level. On re-run, it should preserve or repair:

- the Terraform state KMS key
- the Terraform state S3 bucket
- the required KMS key policy for management-account backend access
- the required S3 bucket policy for management-account backend access

## Prerequisites

1. **STDB org bootstrap completed** — OUs, accounts, and `STDBBootstrapRole` exist in all child accounts.
2. **IAM user created** in the management account with an access key.
3. **IAM user policy** — the user needs only `sts:AssumeRole` permission:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeBootstrapRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/STDBBootstrapRole"
    }
  ]
}
```

4. **Runbook executed** — `Setup-Terraform-Local-Foundation` has been run to create the KMS key, S3 state bucket, and `STDBBootstrapRole` in the management account.
5. **Terraform version** — use Terraform `1.10` or later if `use_lockfile = true` is enabled for the S3 backend.

## Running the Runbook

### Upload the document

```bash
aws ssm create-document \
  --name "Setup-Terraform-Local-Foundation" \
  --document-type "Automation" \
  --document-format YAML \
  --content file://Setup-Terraform-Local-Foundation.yaml \
  --profile orga --region ap-south-1
```

### Execute

```bash
aws ssm start-automation-execution \
  --document-name "Setup-Terraform-Local-Foundation" \
  --parameters '{
    "AutomationAssumeRole": ["arn:aws:iam::<MGMT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>"],
    "StateBucketName": ["stdb-terraform-state-<MGMT_ACCOUNT_ID>"]
  }' \
  --profile orga --region ap-south-1
```

### Check status

```bash
aws ssm get-automation-execution \
  --automation-execution-id "<EXECUTION_ID>" \
  --query 'AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}' \
  --profile orga
```

The outputs will include:
- `discoverContext.AccountId` — your management account ID
- `ensureStateKey.StateKmsKeyArn` — KMS key ARN for state encryption
- `ensureStateKey.UpdatedKeyPolicy` — whether the runbook repaired KMS key-policy drift
- `ensureStateKey.KeyWasCreated` — whether the KMS key was created in this run
- `ensureStateBucket.StateBucketName` — S3 bucket name
- `ensureStateBucket.UpdatedBucketPolicy` — whether the runbook repaired bucket-policy drift
- `ensureStateBucket.BucketWasCreated` — whether the bucket was created in this run
- `ensureBootstrapRoleInManagementAccount.BootstrapRoleArn` — bootstrap role ARN in mgmt account

## AWS CLI Profile Setup

### ~/.aws/credentials

```ini
[stdb]
aws_access_key_id = <YOUR_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY>
```

### ~/.aws/config

```ini
[profile stdb]
region = ap-south-1
output = json

[profile stdb-mgmt]
region = ap-south-1
source_profile = stdb
role_arn = arn:aws:iam::<MGMT_ACCOUNT_ID>:role/STDBBootstrapRole
role_session_name = terraform-mgmt

[profile stdb-security]
region = ap-south-1
source_profile = stdb
role_arn = arn:aws:iam::<SECURITY_ACCOUNT_ID>:role/STDBBootstrapRole
role_session_name = terraform-security

[profile stdb-network]
region = ap-south-1
source_profile = stdb
role_arn = arn:aws:iam::<NETWORK_ACCOUNT_ID>:role/STDBBootstrapRole
role_session_name = terraform-network

[profile stdb-processing]
region = ap-south-1
source_profile = stdb
role_arn = arn:aws:iam::<PROCESSING_ACCOUNT_ID>:role/STDBBootstrapRole
role_session_name = terraform-processing
```

You can verify each profile works with:

```bash
aws sts get-caller-identity --profile stdb-mgmt
```

## Terraform Backend Configuration

All Terraform stacks share the same S3 bucket. Each stack uses a unique `key` to isolate its state file.

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket       = "<BUCKET_NAME>"
    key          = "security-baseline/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "<KMS_KEY_ARN>"
    use_lockfile = true
    role_arn     = "arn:aws:iam::<MGMT_ACCOUNT_ID>:role/STDBBootstrapRole"
  }
}
```

| Field | Value | Notes |
|-------|-------|-------|
| `bucket` | From runbook output `ensureStateBucket.StateBucketName` | |
| `key` | Unique per stack, e.g. `network/terraform.tfstate` | |
| `region` | `ap-south-1` | Or your chosen region |
| `encrypt` | `true` | Server-side encryption with KMS |
| `kms_key_id` | From runbook output `ensureStateKey.StateKmsKeyArn` | |
| `use_lockfile` | `true` | S3-native locking (Terraform 1.10+), no DynamoDB needed |
| `role_arn` | `STDBBootstrapRole` in mgmt account | Backend assumes this role for state access |

Backend access is always through the management-account `STDBBootstrapRole`. Child-account bootstrap roles should not be granted direct access to the shared state bucket or the state KMS key.

If the bucket policy or KMS key policy drifts later, re-running `Setup-Terraform-Local-Foundation` is expected to restore the intended backend-access model.

## Provider Configuration

Each account gets its own provider block. Terraform assumes `STDBBootstrapRole` in the target account for resource creation and account-scoped operations.

### Optional locals example

```hcl
locals {
  region = "ap-south-1"

  accounts = {
    management = "<MGMT_ACCOUNT_ID>"
    security   = "<SECURITY_ACCOUNT_ID>"
    network    = "<NETWORK_ACCOUNT_ID>"
    processing = "<PROCESSING_ACCOUNT_ID>"
    vault      = "<VAULT_ACCOUNT_ID>"
  }
}
```

```hcl
# --- Management account ---
provider "aws" {
  alias  = "management"
  region = local.region
  assume_role {
    role_arn     = "arn:aws:iam::${local.accounts.management}:role/STDBBootstrapRole"
    session_name = "terraform-mgmt"
  }
}

# --- Security account ---
provider "aws" {
  alias  = "security"
  region = local.region
  assume_role {
    role_arn     = "arn:aws:iam::${local.accounts.security}:role/STDBBootstrapRole"
    session_name = "terraform-security"
  }
}

# --- Network account ---
provider "aws" {
  alias  = "network"
  region = local.region
  assume_role {
    role_arn     = "arn:aws:iam::${local.accounts.network}:role/STDBBootstrapRole"
    session_name = "terraform-network"
  }
}

# --- Processing account ---
provider "aws" {
  alias  = "processing"
  region = local.region
  assume_role {
    role_arn     = "arn:aws:iam::${local.accounts.processing}:role/STDBBootstrapRole"
    session_name = "terraform-processing"
  }
}

# --- Vault account ---
provider "aws" {
  alias  = "vault"
  region = local.region
  assume_role {
    role_arn     = "arn:aws:iam::${local.accounts.vault}:role/STDBBootstrapRole"
    session_name = "terraform-vault"
  }
}
```

### Example resource usage

Resources reference the provider explicitly:

```hcl
resource "aws_s3_bucket" "logs" {
  provider = aws.security
  bucket   = "stdb-central-logs-ap-south-1"
}
```

```hcl
resource "aws_ssm_parameter" "example" {
  provider = aws.management
  name     = "/stdb/lz/example"
  type     = "String"
  value    = "example"
}
```

```hcl
resource "aws_kms_key" "vault_key" {
  provider                = aws.vault
  description             = "Vault account key for STDB baseline"
  deletion_window_in_days = 30
}
```

Use the management-account provider only for management-account resources. Use the target-account provider alias for every resource created in a child account.

This means:

- backend access is always through the management-account bootstrap role
- provider access is always through the target-account bootstrap role
- target-account roles do not need direct permissions to the shared state bucket or the state KMS key

## Suggested Directory Structure

```
terraform/
├── backend.tf              # Shared backend config (partial — key set per stack)
├── providers.tf            # All provider blocks with assume_role
├── variables.tf            # Account IDs, region, common variables
├── outputs.tf              # Cross-stack outputs
│
├── organization/           # Org-level resources (SCPs, policies)
│   ├── main.tf
│   └── backend.tf          # key = "organization/terraform.tfstate"
│
├── security-baseline/      # GuardDuty, SecurityHub, CloudTrail
│   ├── main.tf
│   └── backend.tf          # key = "security-baseline/terraform.tfstate"
│
├── networking/             # VPCs, TGW, Route53
│   ├── main.tf
│   └── backend.tf          # key = "networking/terraform.tfstate"
│
└── workloads/              # Per-workload stacks
    ├── processing/
    └── vault/
```

Each subdirectory has its own `backend.tf` with a unique `key`. Run `terraform init` in each directory separately.

## Common Operations

### Initialize a stack

```bash
cd terraform/organization
terraform init
```

### Plan changes

```bash
terraform plan -out=plan.tfplan
```

### Apply changes

```bash
terraform apply plan.tfplan
```

### View state

```bash
terraform state list
terraform state show <resource_address>
```

### Validate configuration

```bash
terraform validate
terraform fmt -check
```

## Security Notes

1. **IAM user permissions are minimal** — only `sts:AssumeRole`. All backend and infrastructure permissions come from `STDBBootstrapRole`.

2. **STDBBootstrapRole uses AdministratorAccess** — this is intentional for the initial landing zone build. Replace with a scoped policy once the landing zone deployment actions are known (tracked in TODO.md).

3. **Credential rotation** — rotate the IAM user access key periodically:
   ```bash
   aws iam create-access-key --user-name <USER_NAME> --profile orga
   # Update ~/.aws/credentials with the new key
   aws iam delete-access-key --user-name <USER_NAME> --access-key-id <OLD_KEY_ID> --profile orga
   ```

4. **State bucket is locked down** — only `STDBBootstrapRole` in the management account and the management account root can access state data. Child-account bootstrap roles are not used for backend access.

5. **KMS access follows the same boundary** — the KMS key used for Terraform state is intended for management-account backend access only, not for child-account bootstrap roles.

6. **S3 native locking** — `use_lockfile = true` prevents concurrent state modifications without requiring a DynamoDB table. Requires Terraform 1.10 or later.
