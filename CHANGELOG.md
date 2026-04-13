# STDB SSM Runbooks - Changelog

All notable changes to the SSM runbook suite are documented in this file.

## [Unreleased] - 2026-04-12

### Added

- **Validate-Config.yaml**: New preflight runbook that validates the org bootstrap config before any provisioning runs. Checks supported config version, required bootstrap fields (managementAccountId, roleName, orgAccessRoleName), OU key/name uniqueness, account name/email uniqueness, and valid targetOuKey references. Fails fast with operator-friendly errors listing all validation failures.

- **Verify-Bootstrap.yaml**: New post-run verification runbook that compares the live AWS Organizations state against the bootstrap config. Checks OU existence, account existence, account OU placement, and bootstrap role assumability. Returns a structured pass/fail report with mismatch details.

- **Bootstrap-STDB-Organization.yaml**: Added `validateConfig` as the first step in the parent orchestrator. Config validation now runs before `discoverRoot`. Added `ValidateConfigDocumentName` parameter for document name override.

- **Account tagging support**: `Create-Accounts.yaml` now reads an optional `accountTags` section from the config and passes tags to the `organizations:CreateAccount` API. The example config (`examples/stdb-org-bootstrap-config.json`) includes sample tags for Environment, ManagedBy, Project, and Lifecycle.

- **Structured step metadata**: All five core runbooks (Ensure-OUs, Create-Accounts, Wait-For-Account-Creation, Move-Accounts-To-OUs, Create-Bootstrap-Role) now emit a `StepMetadata` output containing step name, UTC timestamp, status, counts, and recovery guidance for partial failures.

- **RunId correlation ID**: All runbooks now accept an optional `RunId` parameter. The parent orchestrator (`Bootstrap-STDB-Organization.yaml`) auto-generates a RunId in the format `bootstrap-{YYYYMMDDThhmmss}-{uuid6}` if not provided, and passes it to every child execution. Child runbooks that run standalone also generate their own RunId if none is supplied.

- **Structured JSON logging**: Every `aws:executeScript` step across all 10 runbooks now emits structured JSON log entries via `print()`. Each entry includes `runId`, `step`, `event`, and context-specific fields. When CloudWatch Logs is enabled for SSM Automation, these entries flow to the configured log group for centralized querying.

- **CloudWatch Logs setup**: Added CLI commands in `AWS-CLI-COMMANDS.md` (Section 3) for enabling CloudWatch Logs for SSM Automation `aws:executeScript` output. Includes log group creation with 90-day retention, service setting configuration, and CloudWatch Logs Insights query examples.

- **CloudWatch Logs IAM permissions**: Added `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:DescribeLogGroups`, `logs:DescribeLogStreams`, and `logs:PutLogEvents` to the automation role's bootstrap permissions policy.

- **stdb-key-vault account**: Added `stdb-key-vault` to the bootstrap config under Security OU. This account owns the customer-managed KMS keys used by Prod-hosted logically air-gapped backup vaults. CMK strategy documented in `backup-vault-cmk-strategy.md`.

- **backup-vault-cmk-strategy.md**: New design document capturing the CMK ownership model — STDB owns the keys, Prod receives usage-only permissions. Tracked as decision `DL-012`, status In Review pending POC validation.

- **ROOT-EMAIL-UPDATE.md**: New operating guide for updating root user / primary email addresses on member accounts after creation, using the `account:StartPrimaryEmailUpdate` / `AcceptPrimaryEmailUpdate` API flow from the management account.

- **Secure-Root-Access.yaml**: New standalone runbook that enables centralized root access management for the organization (trusted access for IAM, root credentials management, root sessions) and removes root credentials from all configured child accounts. Uses `sts:AssumeRoot` with `IAMDeleteRootUserCredentials` task policy to delete root passwords, access keys, signing certificates, and MFA devices. Idempotent and safe to re-run. This is intentionally not wired into the parent bootstrap orchestrator because it is a sensitive post-bootstrap hardening control.

### Changed

- **Create-Bootstrap-Role.yaml**: The `assume_into_account` function now retries up to 5 times with exponential backoff on `AccessDenied` / `AccessDeniedException` errors to handle IAM eventual consistency after account creation.

- **Move-Accounts-To-OUs.yaml**: The `move_account` API call now retries up to 3 times with exponential backoff on `ConcurrentModificationException` errors.

- **Wait-For-Account-Creation.yaml**: The polling sleep now includes random jitter (0-10 seconds added to the base 30-second interval) to reduce API contention when multiple accounts are being polled.

### Files Added

- `Validate-Config.yaml`
- `Verify-Bootstrap.yaml`
- `Secure-Root-Access.yaml`
- `ROOT-EMAIL-UPDATE.md`
- `backup-vault-cmk-strategy.md`
- `CHANGELOG.md`

### Files Modified

- `Bootstrap-STDB-Organization.yaml`
- `Create-Bootstrap-Role.yaml`
- `Move-Accounts-To-OUs.yaml`
- `Wait-For-Account-Creation.yaml`
- `Ensure-OUs.yaml`
- `Create-Accounts.yaml`
- `Discover-Root.yaml`
- `Enable-Account-Regions.yaml`
- `Verify-Bootstrap.yaml`
- `Validate-Config.yaml`
- `AWS-CLI-COMMANDS.md`
- `examples/stdb-org-bootstrap-config.json`
- `README.md`
- `RUNBOOK-DEPLOYMENT.md`
- `TODO.md`
