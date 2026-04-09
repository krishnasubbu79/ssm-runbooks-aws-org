# STDB Bootstrap — AWS CLI Commands

This document provides the exact AWS CLI commands to deploy and execute the STDB organization bootstrap runbooks.

Replace the placeholder values before running:

- `<MANAGEMENT_ACCOUNT_ID>` — your AWS management account ID
- `<AUTOMATION_ROLE_NAME>` — the name of the SSM Automation execution role
- `<REGION>` — the AWS region where SSM documents and Parameter Store config reside
- `gmail.com` — your email domain for account root aliases

All commands use `--profile orga`. Ensure this profile is configured in your AWS CLI credentials and points to the management account.

---

## 1. Store the Config in Parameter Store

```bash
aws ssm put-parameter \
  --profile orga \
  --region <REGION> \
  --name "/stdb/org/bootstrap/config" \
  --type "String" \
  --value '{
    "version": "1.0",
    "bootstrap": {
      "roleName": "STDBBootstrapRole",
      "orgAccessRoleName": "OrganizationAccountAccessRole",
      "managementAccountId": "<MANAGEMENT_ACCOUNT_ID>"
    },
    "organizationalUnits": [
      {"key": "security", "name": "Security OU", "parentKey": "root"},
      {"key": "infra", "name": "Infra OU", "parentKey": "root"},
      {"key": "processing", "name": "Processing OU", "parentKey": "root"},
      {"key": "recovery", "name": "Recovery OU", "parentKey": "root"},
      {"key": "vault", "name": "Vault OU", "parentKey": "root"},
      {"key": "suspended", "name": "Suspended OU", "parentKey": "root"}
    ],
    "accounts": [
      {"name": "stdb-log-archive", "email": "aws+stdb-log-archive@yourdomain.com", "targetOuKey": "security"},
      {"name": "stdb-delegated-admin", "email": "aws+stdb-delegated-admin@yourdomain.com", "targetOuKey": "security"},
      {"name": "stdb-network", "email": "aws+stdb-network@yourdomain.com", "targetOuKey": "infra"},
      {"name": "stdb-processing", "email": "aws+stdb-processing@yourdomain.com", "targetOuKey": "processing"},
      {"name": "stdb-recovery", "email": "aws+stdb-recovery@yourdomain.com", "targetOuKey": "recovery"},
      {"name": "stdb-vault", "email": "aws+stdb-vault@yourdomain.com", "targetOuKey": "vault"}
    ]
  }'
```

To verify:

```bash
aws ssm get-parameter \
  --profile orga \
  --region <REGION> \
  --name "/stdb/org/bootstrap/config" \
  --query "Parameter.Value" \
  --output text | python3 -m json.tool
```

To update the config later (requires `--overwrite`):

```bash
aws ssm put-parameter \
  --profile orga \
  --region <REGION> \
  --name "/stdb/org/bootstrap/config" \
  --type "String" \
  --overwrite \
  --value '{ ... updated JSON ... }'
```

---

## Roles Used in This Bootstrap

| Role / User | Where It Lives | Created By | Purpose |
|---|---|---|---|
| `<AUTOMATION_ROLE_NAME>` | Management account | You (Step 2 below) | The IAM role that SSM Automation assumes to execute the runbooks. Trusted by `ssm.amazonaws.com`. |
| `OrganizationAccountAccessRole` | Each child account | AWS automatically during `CreateAccount` | The default cross-account role AWS creates in every new account. The automation role assumes this to reach into child accounts. |
| `STDBBootstrapRole` | Each child account | The `Create-Bootstrap-Role` runbook (Step 7) | A bootstrap role trusted by the management account. Used for landing zone deployment into child accounts. |
| `STDBBootstrapRole` | Management account | The `Setup-Terraform-Local-Foundation` runbook (Step 13) | Same bootstrap role, created in the management account so Terraform has a uniform assume-role pattern across all accounts. |
| `STDBTerraformUser` | Management account | You (Step 13f) | IAM user for local Terraform execution. Has only `sts:AssumeRole` to `STDBBootstrapRole`. |

---

## 2. Create the Automation Execution Role

### 2a. Create the trust policy file

```bash
cat > /tmp/ssm-automation-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ssm.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

### 2b. Create the role

```bash
aws iam create-role \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --assume-role-policy-document file:///tmp/ssm-automation-trust-policy.json \
  --description "SSM Automation execution role for STDB organization bootstrap"
```

### 2c. Create the permissions policy file

```bash
cat > /tmp/ssm-automation-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMParameterAccess",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter"
      ],
      "Resource": "arn:aws:ssm:*:<MANAGEMENT_ACCOUNT_ID>:parameter/stdb/org/bootstrap/*"
    },
    {
      "Sid": "OrganizationsReadAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:ListRoots",
        "organizations:ListOrganizationalUnitsForParent",
        "organizations:ListAccounts",
        "organizations:DescribeCreateAccountStatus",
        "organizations:ListCreateAccountStatus",
        "organizations:ListParents"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OrganizationsWriteAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:CreateOrganizationalUnit",
        "organizations:CreateAccount",
        "organizations:MoveAccount"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AccountRegionAccess",
      "Effect": "Allow",
      "Action": [
        "account:ListRegions",
        "account:EnableRegion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AssumeRoleIntoChildAccounts",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/OrganizationAccountAccessRole"
    },
    {
      "Sid": "IAMBootstrapRoleManagement",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:CreateRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:AttachRolePolicy"
      ],
      "Resource": "arn:aws:iam::*:role/STDBBootstrapRole"
    }
  ]
}
EOF
```

### 2d. Attach the bootstrap permissions policy

```bash
aws iam put-role-policy \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --policy-name "STDBBootstrapAutomationPolicy" \
  --policy-document file:///tmp/ssm-automation-permissions.json
```

### 2e. Create the automation execution permissions policy

The parent orchestrator uses `aws:executeAutomation` to launch child runbooks. This requires `ssm:StartAutomationExecution` and the ability to pass the automation role to the child executions via `iam:PassRole`.

```bash
cat > /tmp/ssm-automation-execution-permissions.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMAutomationExecution",
      "Effect": "Allow",
      "Action": [
        "ssm:StartAutomationExecution",
        "ssm:GetAutomationExecution",
        "ssm:DescribeAutomationExecutions",
        "ssm:DescribeAutomationStepExecutions",
        "ssm:DescribeDocument",
        "ssm:GetDocument"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassRoleToAutomation",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ssm.amazonaws.com"
        }
      }
    }
  ]
}
EOF
```

### 2f. Attach the automation execution permissions policy

```bash
aws iam put-role-policy \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --policy-name "STDBAutomationExecutionPolicy" \
  --policy-document file:///tmp/ssm-automation-execution-permissions.json
```

### 2g. Verify the role

```bash
aws iam get-role \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --query "Role.Arn" \
  --output text
```

Save this ARN — it is the `AutomationAssumeRole` value for all runbook executions.

---

## 3. Upload the SSM Automation Documents

Upload child runbooks first, then the parent orchestrator.

### 3a. Upload child runbooks

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Discover-Root" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Discover-Root.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Ensure-OUs" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Ensure-OUs.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Create-Accounts" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Create-Accounts.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Wait-For-Account-Creation" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Wait-For-Account-Creation.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Move-Accounts-To-OUs" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Move-Accounts-To-OUs.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Create-Bootstrap-Role" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Create-Bootstrap-Role.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Enable-Account-Regions" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Enable-Account-Regions.yaml"
```

### 3b. Upload the parent orchestrator

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Bootstrap-STDB-Organization" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Bootstrap-STDB-Organization.yaml"
```

### 3c. Verify all documents are created

```bash
aws ssm list-documents \
  --profile orga \
  --region <REGION> \
  --filters "Key=Owner,Values=Self" "Key=DocumentType,Values=Automation" \
  --query "DocumentIdentifiers[].Name" \
  --output table
```

### 3d. Updating documents after changes

If you need to update a document after the initial upload, use `update-document` with a new version:

```bash
aws ssm update-document \
  --profile orga \
  --region <REGION> \
  --name "<DOCUMENT_NAME>" \
  --document-format "YAML" \
  --content "file://<DOCUMENT_NAME>.yaml" \
  --document-version "\$LATEST"

aws ssm update-document-default-version \
  --profile orga \
  --region <REGION> \
  --name "<DOCUMENT_NAME>" \
  --document-version "<NEW_VERSION_NUMBER>"
```

---

## 4. Staged Execution — Phase A: Discover Root and Create OUs

### 4a. Discover the organization root

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Discover-Root" \
  --parameters "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>"
```

Check the execution status:

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}" \
  --output json
```

### 4b. Create the OUs

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Ensure-OUs" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 4c. Verify OUs exist

```bash
ROOT_ID=$(aws organizations list-roots --profile orga --query "Roots[0].Id" --output text)

aws organizations list-organizational-units-for-parent \
  --profile orga \
  --parent-id "$ROOT_ID" \
  --query "OrganizationalUnits[].{Name:Name,Id:Id}" \
  --output table
```

---

## 5. Staged Execution — Phase B: Create Accounts

### 5a. Create the accounts

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Create-Accounts" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 5b. Wait for account creation to complete

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Wait-For-Account-Creation" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 5c. Monitor the execution

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}" \
  --output json
```

### 5d. Verify accounts exist

```bash
aws organizations list-accounts \
  --profile orga \
  --query "Accounts[?starts_with(Name,'stdb-')].{Name:Name,Id:Id,Email:Email,Status:Status}" \
  --output table
```

---

## 6. Staged Execution — Phase C: Move Accounts to OUs

### 6a. Move accounts

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Move-Accounts-To-OUs" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 6b. Verify account placement

```bash
ROOT_ID=$(aws organizations list-roots --profile orga --query "Roots[0].Id" --output text)

for OU_ID in $(aws organizations list-organizational-units-for-parent \
  --profile orga \
  --parent-id "$ROOT_ID" \
  --query "OrganizationalUnits[].Id" \
  --output text); do

  OU_NAME=$(aws organizations list-organizational-units-for-parent \
    --profile orga \
    --parent-id "$ROOT_ID" \
    --query "OrganizationalUnits[?Id=='$OU_ID'].Name" \
    --output text)

  echo "--- $OU_NAME ($OU_ID) ---"

  aws organizations list-accounts-for-parent \
    --profile orga \
    --parent-id "$OU_ID" \
    --query "Accounts[].{Name:Name,Id:Id}" \
    --output table
done
```

---

## 7. Staged Execution — Phase D: Create Bootstrap Role

### 7a. Create the bootstrap role in each account

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Create-Bootstrap-Role" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 7b. Verify you can assume the bootstrap role

Test with one account:

```bash
aws sts assume-role \
  --profile orga \
  --role-arn "arn:aws:iam::<CHILD_ACCOUNT_ID>:role/STDBBootstrapRole" \
  --role-session-name "bootstrap-verify" \
  --query "Credentials.AccessKeyId" \
  --output text
```

Test all accounts:

```bash
for ACCOUNT_ID in $(aws organizations list-accounts \
  --profile orga \
  --query "Accounts[?starts_with(Name,'stdb-')].Id" \
  --output text); do

  echo -n "Account $ACCOUNT_ID: "
  aws sts assume-role \
    --profile orga \
    --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/STDBBootstrapRole" \
    --role-session-name "bootstrap-verify" \
    --query "Credentials.AccessKeyId" \
    --output text 2>/dev/null && echo "OK" || echo "FAILED"
done
```

---

## 8. Staged Execution — Phase E: Enable Opt-In Regions

### 8a. Enable regions

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Enable-Account-Regions" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config,RegionNames=ap-south-2"
```

### 8b. Verify region status

```bash
for ACCOUNT_ID in $(aws organizations list-accounts \
  --profile orga \
  --query "Accounts[?starts_with(Name,'stdb-')].Id" \
  --output text); do

  echo -n "Account $ACCOUNT_ID ap-south-2: "
  aws account get-region-opt-status \
    --profile orga \
    --region-name "ap-south-2" \
    --query "RegionOptStatus" \
    --output text 2>/dev/null || echo "CHECK MANUALLY"
done
```

---

## 9. Full Orchestrated Execution

Once the staged rollout has been verified, subsequent runs can use the parent orchestrator:

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Bootstrap-STDB-Organization" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

Monitor progress:

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,CurrentStep:CurrentStepName,Outputs:Outputs}" \
  --output json
```

List recent automation executions:

```bash
aws ssm describe-automation-executions \
  --profile orga \
  --region <REGION> \
  --filters "Key=DocumentNamePrefix,Values=Bootstrap-STDB" \
  --query "AutomationExecutionMetadataList[].{Id:AutomationExecutionId,Status:AutomationExecutionStatus,Start:ExecutionStartTime,Document:DocumentName}" \
  --output table
```

---

## 10. Teardown — Upload Teardown Runbooks

The teardown runbooks live in `account-deletion/`. Upload them the same way as the bootstrap runbooks.

```bash
for doc in Remove-Bootstrap-Roles Move-Accounts-To-Root Close-Accounts Delete-OUs; do
  aws ssm create-document \
    --profile orga \
    --region <REGION> \
    --name "$doc" \
    --document-type "Automation" \
    --document-format "YAML" \
    --content "file://account-deletion/${doc}.yaml"
done

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Teardown-STDB-Organization" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://account-deletion/Teardown-STDB-Organization.yaml"
```

### Add teardown permissions to the automation role

The automation role needs additional permissions for teardown operations:

```bash
cat > /tmp/ssm-automation-teardown-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OrganizationsTeardownAccess",
      "Effect": "Allow",
      "Action": [
        "organizations:CloseAccount",
        "organizations:DeleteOrganizationalUnit",
        "organizations:ListChildren"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMBootstrapRoleTeardown",
      "Effect": "Allow",
      "Action": [
        "iam:ListAttachedRolePolicies",
        "iam:DetachRolePolicy",
        "iam:DeleteRole"
      ],
      "Resource": "arn:aws:iam::*:role/STDBBootstrapRole"
    }
  ]
}
EOF

aws iam put-role-policy \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --policy-name "STDBTeardownAutomationPolicy" \
  --policy-document file:///tmp/ssm-automation-teardown-permissions.json
```

---

## 11. Teardown — Staged Execution

Run teardown in phases, verifying after each step.

### 11a. Remove bootstrap roles from child accounts

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Remove-Bootstrap-Roles" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 11b. Move accounts back to root

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Move-Accounts-To-Root" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

Verify all accounts are at root:

```bash
ROOT_ID=$(aws organizations list-roots --profile orga --query "Roots[0].Id" --output text)

aws organizations list-accounts-for-parent \
  --profile orga \
  --parent-id "$ROOT_ID" \
  --query "Accounts[?starts_with(Name,'stdb-')].{Name:Name,Id:Id}" \
  --output table
```

### 11c. Close the accounts

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Close-Accounts" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

Verify account statuses:

```bash
aws organizations list-accounts \
  --profile orga \
  --query "Accounts[?starts_with(Name,'stdb-')].{Name:Name,Id:Id,Status:Status}" \
  --output table
```

### 11d. Delete the OUs

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Delete-OUs" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

Verify OUs are gone:

```bash
ROOT_ID=$(aws organizations list-roots --profile orga --query "Roots[0].Id" --output text)

aws organizations list-organizational-units-for-parent \
  --profile orga \
  --parent-id "$ROOT_ID" \
  --query "OrganizationalUnits[].{Name:Name,Id:Id}" \
  --output table
```

---

## 12. Teardown — Full Orchestrated Execution

Once you are confident the staged teardown works, subsequent runs can use the parent orchestrator:

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Teardown-STDB-Organization" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

---

## 13. Setup Terraform Local Foundation

This section provisions the Terraform prerequisites for running Terraform from your local laptop against all STDB accounts. The runbook creates:

- A KMS key for state encryption (with an explicit key policy granting `STDBBootstrapRole` usage)
- An S3 bucket for Terraform remote state (versioned, encrypted, hardened bucket policy)
- `STDBBootstrapRole` in the management account (the bootstrap runbooks only create it in child accounts)

After this runbook completes, you can run `terraform plan` and `terraform apply` from your laptop. Terraform assumes `STDBBootstrapRole` in every account — including the management account — for a uniform execution model.

On re-run, the runbook is intended to be convergent at both the resource and policy level. If the Terraform state KMS key policy or S3 bucket policy drifts, the runbook should repair it.

### 13a. Add Terraform foundation permissions to the automation role

**You must run this step before executing the runbook.** The automation role created in Step 2 does not include the full set of KMS, S3 bucket-policy, or management-account IAM role permissions required by the convergent local-foundation runbook. This step adds them as a separate inline policy.

If the runbook fails with `AccessDeniedException` on `kms:GetKeyPolicy`, `kms:PutKeyPolicy`, `s3:GetBucketPolicy`, or similar, this policy is missing or incomplete.

```bash
cat > /tmp/ssm-terraform-foundation-permissions.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KMSKeyManagement",
      "Effect": "Allow",
      "Action": [
        "kms:CreateKey",
        "kms:CreateAlias",
        "kms:DescribeKey",
        "kms:ListAliases",
        "kms:GetKeyPolicy",
        "kms:PutKeyPolicy"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3BucketManagement",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:ListBucket",
        "s3:PutBucketVersioning",
        "s3:PutEncryptionConfiguration",
        "s3:GetBucketPolicy",
        "s3:PutBucketPolicy",
        "s3:PutBucketPublicAccessBlock"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMBootstrapRoleInManagementAccount",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:CreateRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:AttachRolePolicy",
        "iam:ListAttachedRolePolicies"
      ],
      "Resource": "arn:aws:iam::*:role/STDBBootstrapRole"
    }
  ]
}
EOF

aws iam put-role-policy \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --policy-name "STDBTerraformFoundationPolicy" \
  --policy-document file:///tmp/ssm-terraform-foundation-permissions.json
```

If you already attached an earlier version of this inline policy before the convergence change, rerun the `aws iam put-role-policy` command above to overwrite it with the updated permissions.

Verify the policy is attached:

```bash
aws iam list-role-policies \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --output table
```

You should see these inline policies:
- `STDBBootstrapAutomationPolicy` (from Step 2d)
- `STDBAutomationExecutionPolicy` (from Step 2f)
- `STDBTerraformFoundationPolicy` (from Step 13a above)

### 13b. Upload the runbook

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Setup-Terraform-Local-Foundation" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Setup-Terraform-Local-Foundation.yaml"
```

### 13c. Execute the runbook

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Setup-Terraform-Local-Foundation" \
  --parameters '{
    "AutomationAssumeRole": ["arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>"],
    "StateBucketName": ["stdb-terraform-state-<MANAGEMENT_ACCOUNT_ID>"]
  }'
```

### 13d. Monitor execution

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}" \
  --output json
```

### 13e. Retrieve the outputs

Save these values — you will need them for your Terraform backend and provider configuration:

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.Outputs" \
  --output json
```

Expected outputs:

| Output | Description |
|--------|-------------|
| `discoverContext.AccountId` | Management account ID |
| `discoverContext.Region` | AWS region |
| `ensureStateKey.StateKmsKeyArn` | KMS key ARN for state encryption |
| `ensureStateKey.UpdatedKeyPolicy` | `true` if the runbook repaired the KMS key policy |
| `ensureStateKey.KeyWasCreated` | `true` if the KMS key was created in this run |
| `ensureStateBucket.StateBucketName` | S3 state bucket name |
| `ensureStateBucket.UpdatedBucketPolicy` | `true` if the runbook repaired the bucket policy |
| `ensureStateBucket.BucketWasCreated` | `true` if the bucket was created in this run |
| `ensureBootstrapRoleInManagementAccount.BootstrapRoleArn` | Bootstrap role ARN in management account |

### 13f. Create the Terraform IAM user

Create a dedicated IAM user for Terraform. This user only needs `sts:AssumeRole` — all actual permissions come from `STDBBootstrapRole`.

```bash
aws iam create-user \
  --profile orga \
  --user-name STDBTerraformUser

cat > /tmp/terraform-user-policy.json << 'EOF'
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
EOF

aws iam put-user-policy \
  --profile orga \
  --user-name STDBTerraformUser \
  --policy-name STDBTerraformUserPolicy \
  --policy-document file:///tmp/terraform-user-policy.json

aws iam create-access-key \
  --profile orga \
  --user-name STDBTerraformUser
```

Save the `AccessKeyId` and `SecretAccessKey` from the output.

### 13g. Configure AWS CLI profiles

Add the IAM user credentials to `~/.aws/credentials`:

```ini
[stdb]
aws_access_key_id = <ACCESS_KEY_ID>
aws_secret_access_key = <SECRET_ACCESS_KEY>
```

Add per-account profiles to `~/.aws/config`:

```ini
[profile stdb]
region = ap-south-1
output = json

[profile stdb-mgmt]
region = ap-south-1
source_profile = stdb
role_arn = arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/STDBBootstrapRole
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

Verify the profiles work:

```bash
aws sts get-caller-identity --profile stdb-mgmt
aws sts get-caller-identity --profile stdb-security
```

### 13h. Terraform backend configuration

Use this in your Terraform code. Each stack uses a unique `key`:

```hcl
terraform {
  backend "s3" {
    bucket       = "<BUCKET_NAME>"
    key          = "<stack-name>/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "<KMS_KEY_ARN>"
    use_lockfile = true
    role_arn     = "arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/STDBBootstrapRole"
  }
}
```

### 13i. Terraform provider configuration

Each account gets its own provider. Terraform assumes `STDBBootstrapRole` in the target account:

```hcl
provider "aws" {
  alias  = "management"
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/STDBBootstrapRole"
    session_name = "terraform-mgmt"
  }
}

provider "aws" {
  alias  = "security"
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::<SECURITY_ACCOUNT_ID>:role/STDBBootstrapRole"
    session_name = "terraform-security"
  }
}

provider "aws" {
  alias  = "network"
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::<NETWORK_ACCOUNT_ID>:role/STDBBootstrapRole"
    session_name = "terraform-network"
  }
}

provider "aws" {
  alias  = "processing"
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::<PROCESSING_ACCOUNT_ID>:role/STDBBootstrapRole"
    session_name = "terraform-processing"
  }
}
```

Resources reference the provider explicitly:

```hcl
resource "aws_s3_bucket" "logs" {
  provider = aws.security
  bucket   = "stdb-central-logs"
}
```

### 13j. Verify end-to-end

```bash
cd your-terraform-directory/
terraform init
terraform plan
```

If `terraform init` succeeds, the backend (S3 + KMS) and credentials are working. If `terraform plan` succeeds, the provider assume-role chain is working.

---

## 14. Cleanup Commands (Reference Only)

These commands are provided for reference. Use with caution.

### Delete all SSM documents

```bash
for doc in Discover-Root Ensure-OUs Create-Accounts Wait-For-Account-Creation \
  Move-Accounts-To-OUs Create-Bootstrap-Role Enable-Account-Regions \
  Bootstrap-STDB-Organization \
  Remove-Bootstrap-Roles Move-Accounts-To-Root Close-Accounts Delete-OUs \
  Teardown-STDB-Organization \
  Setup-Terraform-Local-Foundation; do

  aws ssm delete-document --profile orga --region <REGION> --name "$doc" 2>/dev/null && \
    echo "Deleted: $doc" || echo "Not found: $doc"
done
```

### Delete the config parameter

```bash
aws ssm delete-parameter \
  --profile orga \
  --region <REGION> \
  --name "/stdb/org/bootstrap/config"
```

### Delete the Terraform user

```bash
aws iam delete-user-policy \
  --profile orga \
  --user-name STDBTerraformUser \
  --policy-name STDBTerraformUserPolicy

# List and delete access keys first
aws iam list-access-keys \
  --profile orga \
  --user-name STDBTerraformUser \
  --query "AccessKeyMetadata[].AccessKeyId" \
  --output text | xargs -I {} \
  aws iam delete-access-key --profile orga --user-name STDBTerraformUser --access-key-id {}

aws iam delete-user \
  --profile orga \
  --user-name STDBTerraformUser
```

### Delete the automation role

```bash
for policy in STDBBootstrapAutomationPolicy STDBAutomationExecutionPolicy \
  STDBTeardownAutomationPolicy STDBTerraformFoundationPolicy; do
  aws iam delete-role-policy \
    --profile orga \
    --role-name <AUTOMATION_ROLE_NAME> \
    --policy-name "$policy" 2>/dev/null && \
    echo "Deleted policy: $policy" || echo "Not found: $policy"
done

aws iam delete-role \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME>
```
