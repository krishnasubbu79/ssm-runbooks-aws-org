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
      {"name": "stdb-key-vault", "email": "aws+stdb-key-vault@yourdomain.com", "targetOuKey": "security"},
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
| `<AUTOMATION_ROLE_NAME>` | Management account | You (Step 2 below) | The IAM role that SSM Automation assumes for all bootstrap runbooks. Trusted by `ssm.amazonaws.com`. No root permissions. |
| `STDBSecureRootRole` | Management account | You (Step 2e below) | Dedicated role used **only** by `Secure-Root-Access`. Holds `sts:AssumeRoot` scoped to `IAMDeleteRootUserCredentials`. Isolated to prevent any other automation inheriting root-level access. |
| `OrganizationAccountAccessRole` | Each child account | AWS automatically during `CreateAccount` | The default cross-account role AWS creates in every new account. The automation role assumes this to reach into child accounts. |
| `STDBBootstrapRole` | Each child account | The `Create-Bootstrap-Role` runbook (Step 8) | A bootstrap role trusted by the management account. Used for landing zone deployment into child accounts. |
| `STDBBootstrapRole` | Management account | The `Setup-Terraform-Local-Foundation` runbook (Step 15) | Same bootstrap role, created in the management account so Terraform has a uniform assume-role pattern across all accounts. |
| `STDBTerraformUser` | Management account | You (Step 15f) | IAM user for local Terraform execution. Has only `sts:AssumeRole` to `STDBBootstrapRole`. |

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
    },
    {
      "Sid": "CloudWatchLogsDiscovery",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogsWriteAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:<REGION>:<MANAGEMENT_ACCOUNT_ID>:log-group:/stdb/ssm/automation/executeScript",
        "arn:aws:logs:<REGION>:<MANAGEMENT_ACCOUNT_ID>:log-group:/stdb/ssm/automation/executeScript:log-stream:*"
      ]
    },
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

---

## 2e. Create the Secure Root Role

`Secure-Root-Access` uses a dedicated role (`STDBSecureRootRole`) that is separate from the main bootstrap automation role. This isolates `sts:AssumeRoot` — the most powerful permission in the org — so that no other runbook or automation can accidentally inherit it. The bootstrap role has no root permissions at all.

### 2e. Create the STDBSecureRootRole

```bash
aws iam create-role \
  --profile orga \
  --role-name "STDBSecureRootRole" \
  --assume-role-policy-document file:///tmp/ssm-automation-trust-policy.json \
  --description "Dedicated SSM Automation role for Secure-Root-Access runbook only. Holds sts:AssumeRoot scoped to IAMDeleteRootUserCredentials."
```

### 2f. Create the secure root permissions policy file

```bash
cat > /tmp/ssm-secure-root-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMParameterAccess",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter"],
      "Resource": "arn:aws:ssm:*:<MANAGEMENT_ACCOUNT_ID>:parameter/stdb/org/bootstrap/*"
    },
    {
      "Sid": "OrganizationsReadForAccountResolution",
      "Effect": "Allow",
      "Action": ["organizations:ListAccounts"],
      "Resource": "*"
    },
    {
      "Sid": "CentralizedRootAccessManagement",
      "Effect": "Allow",
      "Action": [
        "organizations:EnableAWSServiceAccess",
        "organizations:ListAWSServiceAccessForOrganization",
        "iam:EnableOrganizationsRootCredentialsManagement",
        "iam:EnableOrganizationsRootSessions",
        "iam:ListOrganizationsFeatures"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AssumeRootForCredentialRemoval",
      "Effect": "Allow",
      "Action": "sts:AssumeRoot",
      "Resource": "arn:aws:iam::*:root",
      "Condition": {
        "StringEquals": {
          "sts:TaskPolicyArn": "arn:aws:iam::aws:policy/root-task/IAMDeleteRootUserCredentials"
        }
      }
    },
    {
      "Sid": "CloudWatchLogsWriteAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:<REGION>:<MANAGEMENT_ACCOUNT_ID>:log-group:/stdb/ssm/automation/executeScript",
        "arn:aws:logs:<REGION>:<MANAGEMENT_ACCOUNT_ID>:log-group:/stdb/ssm/automation/executeScript:log-stream:*"
      ]
    }
  ]
}
EOF
```

### 2g. Attach the secure root permissions policy

```bash
aws iam put-role-policy \
  --profile orga \
  --role-name "STDBSecureRootRole" \
  --policy-name "STDBSecureRootPolicy" \
  --policy-document file:///tmp/ssm-secure-root-permissions.json
```

---

### 2h. Create the automation execution permissions policy

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
      "Resource": [
        "arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>",
        "arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/STDBSecureRootRole"
      ],
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

### 2i. Attach the automation execution permissions policy to both roles

```bash
aws iam put-role-policy \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --policy-name "STDBAutomationExecutionPolicy" \
  --policy-document file:///tmp/ssm-automation-execution-permissions.json

aws iam put-role-policy \
  --profile orga \
  --role-name "STDBSecureRootRole" \
  --policy-name "STDBAutomationExecutionPolicy" \
  --policy-document file:///tmp/ssm-automation-execution-permissions.json
```

### 2j. Verify both roles

```bash
aws iam get-role \
  --profile orga \
  --role-name <AUTOMATION_ROLE_NAME> \
  --query "Role.Arn" \
  --output text

aws iam get-role \
  --profile orga \
  --role-name "STDBSecureRootRole" \
  --query "Role.Arn" \
  --output text
```

Save both ARNs. The bootstrap role ARN is the `AutomationAssumeRole` for all bootstrap runbooks. The `STDBSecureRootRole` ARN is used exclusively for the `Secure-Root-Access` runbook.

---

## 3. Enable CloudWatch Logs for SSM Automation

SSM Automation `aws:executeScript` steps can send their `print()` output to CloudWatch Logs. This is an account-level service setting — once enabled, all automation executions in the account will log to the configured log group.

The runbooks use structured JSON logging with a correlation ID (`RunId`) so you can trace a complete bootstrap run across all child executions using CloudWatch Logs Insights.

SSM Automation creates the log streams from the `aws:executeScript` step output. The runbook Python code does not create CloudWatch log streams directly. The `AutomationAssumeRole` therefore needs CloudWatch Logs permissions that cover both the log group ARN and the log stream ARN (`:log-stream:*`), as shown in the role policy in Step 2.

### 3a. Create the CloudWatch log group

```bash
aws logs create-log-group \
  --profile orga \
  --region <REGION> \
  --log-group-name "/stdb/ssm/automation/executeScript"

aws logs put-retention-policy \
  --profile orga \
  --region <REGION> \
  --log-group-name "/stdb/ssm/automation/executeScript" \
  --retention-in-days 90
```

### 3b. Enable SSM Automation logging to CloudWatch

These two `update-service-setting` calls configure the account-level settings. The first enables CloudWatch logging for `aws:executeScript`; the second sets the target log group.

```bash
aws ssm update-service-setting \
  --profile orga \
  --region <REGION> \
  --setting-id "arn:aws:ssm:<REGION>:<MANAGEMENT_ACCOUNT_ID>:servicesetting/ssm/automation/customer-script-log-destination" \
  --setting-value "CloudWatch"

aws ssm update-service-setting \
  --profile orga \
  --region <REGION> \
  --setting-id "arn:aws:ssm:<REGION>:<MANAGEMENT_ACCOUNT_ID>:servicesetting/ssm/automation/customer-script-log-group-name" \
  --setting-value "/stdb/ssm/automation/executeScript"
```

### 3c. Verify the service settings

```bash
aws ssm get-service-setting \
  --profile orga \
  --region <REGION> \
  --setting-id "arn:aws:ssm:<REGION>:<MANAGEMENT_ACCOUNT_ID>:servicesetting/ssm/automation/customer-script-log-destination" \
  --query "ServiceSetting.SettingValue" \
  --output text

aws ssm get-service-setting \
  --profile orga \
  --region <REGION> \
  --setting-id "arn:aws:ssm:<REGION>:<MANAGEMENT_ACCOUNT_ID>:servicesetting/ssm/automation/customer-script-log-group-name" \
  --query "ServiceSetting.SettingValue" \
  --output text
```

Expected output: `CloudWatch` and `/stdb/ssm/automation/executeScript`.

### 3d. Verify the log group exists

```bash
aws logs describe-log-groups \
  --profile orga \
  --region <REGION> \
  --log-group-name-prefix "/stdb/ssm/automation" \
  --query "logGroups[].{Name:logGroupName,RetentionDays:retentionInDays}" \
  --output table
```

### 3e. Querying logs with CloudWatch Logs Insights

After a bootstrap run, use the RunId to trace all log entries across child executions:

```bash
aws logs start-query \
  --profile orga \
  --region <REGION> \
  --log-group-name "/stdb/ssm/automation/executeScript" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like "bootstrap-" | sort @timestamp asc | limit 200'
```

For a specific RunId:

```bash
aws logs start-query \
  --profile orga \
  --region <REGION> \
  --log-group-name "/stdb/ssm/automation/executeScript" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like "<RUN_ID>" | sort @timestamp asc | limit 500'
```

Retrieve the query results:

```bash
aws logs get-query-results \
  --profile orga \
  --region <REGION> \
  --query-id <QUERY_ID>
```

---

## 4. Upload the SSM Automation Documents

Upload child runbooks first, then the parent orchestrator.

### 4a. Upload child runbooks

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Validate-Config" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Validate-Config.yaml"

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

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Verify-Bootstrap" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Verify-Bootstrap.yaml"

aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Secure-Root-Access" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Secure-Root-Access.yaml"
```

### 4b. Upload the parent orchestrator

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Bootstrap-STDB-Organization" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Bootstrap-STDB-Organization.yaml"
```

### 4c. Verify all documents are created

```bash
aws ssm list-documents \
  --profile orga \
  --region <REGION> \
  --filters "Key=Owner,Values=Self" "Key=DocumentType,Values=Automation" \
  --query "DocumentIdentifiers[].Name" \
  --output table
```

### 4d. Updating documents after changes

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

## 5. Staged Execution — Phase A: Discover Root and Create OUs

### 5a. Discover the organization root

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

### 5b. Create the OUs

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

## 6. Staged Execution — Phase B: Create Accounts

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

## 7. Staged Execution — Phase C: Move Accounts to OUs

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

## 8. Staged Execution — Phase D: Create Bootstrap Role

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

## 9. Staged Execution — Phase E: Secure Root Access

### 9a. Enable centralized root access and remove root credentials

> **Note:** This runbook uses `STDBSecureRootRole` — not the bootstrap automation role. The `sts:AssumeRoot` permission is isolated to this dedicated role and is not present on the main automation role.

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Secure-Root-Access" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/STDBSecureRootRole,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 9b. Verify the execution

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}" \
  --output json
```

### 9c. Verify centralized root features are enabled

```bash
aws iam list-organizations-features \
  --profile orga \
  --output json
```

Expected output should include both `RootCredentialsManagement` and `RootSessions` in the `EnabledFeatures` list.

---

## 10. Staged Execution — Phase F: Enable Opt-In Regions

### 10a. Enable regions

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Enable-Account-Regions" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config,RegionNames=ap-south-2"
```

### 10b. Verify region status

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

## 11. Full Orchestrated Execution

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

## 12. Teardown — Upload Teardown Runbooks

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

## 13. Teardown — Staged Execution

Run teardown in phases, verifying after each step.

### 13a. Remove bootstrap roles from child accounts

```bash
aws ssm start-automation-execution \
  --profile orga \
  --region <REGION> \
  --document-name "Remove-Bootstrap-Roles" \
  --parameters \
    "AutomationAssumeRole=arn:aws:iam::<MANAGEMENT_ACCOUNT_ID>:role/<AUTOMATION_ROLE_NAME>,ConfigParameterName=/stdb/org/bootstrap/config"
```

### 13b. Move accounts back to root

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

### 13c. Close the accounts

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

### 13d. Delete the OUs

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

## 14. Teardown — Full Orchestrated Execution

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

## 15. Setup Terraform Local Foundation

This section provisions the Terraform prerequisites for running Terraform from your local laptop against all STDB accounts. The runbook creates:

- A KMS key for state encryption (with an explicit key policy granting `STDBBootstrapRole` usage)
- An S3 bucket for Terraform remote state (versioned, encrypted, hardened bucket policy)
- `STDBBootstrapRole` in the management account (the bootstrap runbooks only create it in child accounts)

After this runbook completes, you can run `terraform plan` and `terraform apply` from your laptop. Terraform assumes `STDBBootstrapRole` in every account — including the management account — for a uniform execution model.

On re-run, the runbook is intended to be convergent at both the resource and policy level. If the Terraform state KMS key policy or S3 bucket policy drifts, the runbook should repair it.

### 15a. Add Terraform foundation permissions to the automation role

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
- `STDBTerraformFoundationPolicy` (from Step 15a above)

### 15b. Upload the runbook

```bash
aws ssm create-document \
  --profile orga \
  --region <REGION> \
  --name "Setup-Terraform-Local-Foundation" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content "file://Setup-Terraform-Local-Foundation.yaml"
```

### 15c. Execute the runbook

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

### 15d. Monitor execution

```bash
aws ssm get-automation-execution \
  --profile orga \
  --region <REGION> \
  --automation-execution-id <EXECUTION_ID> \
  --query "AutomationExecution.{Status:AutomationExecutionStatus,Outputs:Outputs}" \
  --output json
```

### 15e. Retrieve the outputs

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

### 15f. Create the Terraform IAM user

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

### 15g. Configure AWS CLI profiles

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

### 15h. Terraform backend configuration

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

### 15i. Terraform provider configuration

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

### 15j. Verify end-to-end

```bash
cd your-terraform-directory/
terraform init
terraform plan
```

If `terraform init` succeeds, the backend (S3 + KMS) and credentials are working. If `terraform plan` succeeds, the provider assume-role chain is working.

---

## 16. Cleanup Commands (Reference Only)

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
