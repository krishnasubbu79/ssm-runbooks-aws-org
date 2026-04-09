# STDB Bootstrap TODO

This file tracks the follow-on work needed to harden the current STDB bootstrap implementation from a working foundation into a production-grade operating model.

## Security Hardening

- [ ] Replace `AdministratorAccess` on `STDBBootstrapRole` with a scoped bootstrap policy aligned to the actual landing zone deployment actions.
- [ ] Review and tighten the trust policy for `STDBBootstrapRole` if additional controls such as explicit principal conditions are needed.
- [ ] Split bootstrap and teardown IAM permissions so non-destructive automation does not carry unnecessary destructive rights.
- [ ] Review whether `AutomationAssumeRole` should be separated by function, for example bootstrap role, teardown role, and region-enable role.
- [ ] Re-add `DenyDeleteBucketAndPolicy` statement to the S3 state bucket policy in `Setup-Terraform-Local-Foundation.yaml`. Currently removed for testing flexibility. For production, add a Deny statement for `s3:DeleteBucket` and `s3:DeleteBucketPolicy` scoped to the authorized principals (not `Principal: "*"`) to remain compatible with S3 Block Public Access at account level.

## Input and Config Validation

- [ ] Add a dedicated preflight runbook to validate permissions, config access, KMS decrypt capability, and required AWS service setup (such as trusted access for Account Management) before bootstrap execution.
- [ ] Add explicit schema validation for the Parameter Store config before any provisioning steps run.
- [ ] Validate that OU keys are unique.
- [ ] Validate that OU names are unique within the configured scope.
- [ ] Validate that account names are unique.
- [ ] Validate that account emails are unique.
- [ ] Validate that `targetOuKey` values always reference a defined OU key.
- [ ] Validate that the config `version` is supported before continuing.
- [ ] Add preflight checks for required config fields such as `managementAccountId`, `roleName`, and `orgAccessRoleName`.
- [ ] Add dry-run mode that reports what would be created, moved, or changed without taking any action. This is a significant safety net for production operators.

## State and Auditability

- [ ] Add durable execution-state recording for bootstrap runs.
- [ ] Decide where execution state should live: `S3`, `DynamoDB`, or a structured `Parameter Store` path.
- [ ] Persist key outputs such as OU IDs, account IDs, move results, and bootstrap-role creation status.
- [ ] Define an audit format for run outcomes so operators can answer what changed in a given execution.

## Observability and Verification

- [ ] Add a post-run verification runbook for OUs, accounts, account placement, and bootstrap-role presence.
- [ ] Add a post-run verification step for opt-in Region status where Region enablement is used.
- [ ] Improve structured outputs across the runbooks so execution results are easier to consume operationally.
- [ ] Decide whether CloudWatch logging or a central log destination should be added for automation execution analysis.
- [ ] Add execution outcome notifications via SNS or EventBridge so operators are alerted on success, failure, or partial failure without needing to poll SSM execution status.

## Reliability and Resilience

- [ ] Add retry and backoff handling around eventual consistency points such as `sts:AssumeRole`, IAM role propagation, and account follow-up operations.
- [ ] Review timeout values for long-running operations such as account creation and Region enablement. Note: `aws:executeScript` has a hard 600-second runtime limit — long-polling designs must be split across multiple steps.
- [ ] Add clearer recovery guidance for partial failures, especially where some accounts succeed and others fail.
- [ ] Decide whether failed executions should emit a standard operational signal or notification.

## Governance and Controls

- [ ] Add explicit approval steps for sensitive automation flows such as account creation and teardown.
- [ ] Define when approvals are required versus when unattended execution is acceptable.
- [ ] Design and implement foundational SCPs on OUs — at minimum: deny unapproved Regions, deny leaving the organization, protect the log-archive account from tampering, and restrict root user actions in child accounts.
- [ ] Define the SCP attachment model — which SCPs apply at root level versus per-OU.
- [ ] Define the sequencing rules so SCP-based Region restrictions are applied only after Region enablement is confirmed complete.
- [ ] Document SCP exceptions or escape hatches needed during initial landing zone deployment.

## Account Metadata and Lifecycle

- [ ] Define an account tagging model for all created accounts.
- [ ] Add tags such as purpose, environment, lifecycle, and creation source.
- [ ] Decide how temporary or recovery-created accounts should be reviewed and eventually moved to `Suspended OU`.
- [ ] Define the lifecycle metadata expected for on-demand recovery account creation.

## Runbook Model

- [ ] Decide whether `Enable-Account-Regions` should remain separate permanently or later be wrapped by a higher-level orchestrator.
- [ ] Review whether the parent bootstrap runbook should expose outputs more explicitly for downstream automation.

## Documentation

- [ ] Keep the deployment guide aligned with the actual runbook behavior whenever the runbooks change.
- [ ] Keep the CLI guide aligned with the current runbook set and execution model.
- [ ] Document the centralised root email operating model in the final platform documentation set.
- [ ] Document the operational model for recovery account requests once that flow is implemented.

## Nice To Have

- [ ] Add a verification script or runbook that compares desired config state to actual AWS Organizations state and reports drift.
- [ ] Add a simple release/versioning process for the SSM documents.
- [ ] Add a change log for runbook revisions and document version promotions.
