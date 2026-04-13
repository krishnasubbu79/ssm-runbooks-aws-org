# STDB / Data Bunker AWS Architecture Validation Workshop

## Workshop Purpose

This document proposes a three-day AWS architecture validation workshop for the STDB / Data Bunker design.

The key design constraint is:

- AWS Backup logically air-gapped vaults must reside inside the Production Organization.

The intended security model is:

- STDB acts as the cyber-recovery governance boundary.
- Prod Vault OU owns the AWS Backup logically air-gapped vault.
- STDB owns customer-managed KMS keys through `stdb-key-vault`.
- Prod receives usage-only KMS permissions required for AWS Backup operations.

The workshop should validate this model with AWS and turn open architecture questions into actionable design decisions and POC work.

## Workshop Objectives

By the end of the workshop, the team should have:

1. validated the Vault OU-in-Prod control model
2. confirmed the STDB-owned CMK approach
3. identified AWS service limitations and supported patterns
4. agreed the cross-organization backup, copy, restore, and monitoring model
5. defined POC scope, success criteria, and test account layout
6. captured decisions, risks, AWS follow-ups, and implementation next steps

## Suggested AWS Attendees

Ask AWS to include specialists or SMEs for:

- AWS Backup
- AWS KMS
- AWS Organizations / Control Tower
- IAM
- AWS RAM
- GuardDuty
- Security Hub
- AWS Config
- CloudTrail
- AWS Step Functions / EventBridge / SSM Automation
- Storage / cyber-recovery architecture

## Pre-Read Material

Suggested pre-read for AWS:

- original STDB / Data Bunker architecture diagram
- [aws-review-questionnaire-vault-ou.md](/Users/krishnansubramanian/Documents/New%20project/LZ/aws-review-questionnaire-vault-ou.md)
- [backup-vault-cmk-strategy.md](/Users/krishnansubramanian/Documents/New%20project/ssm/backup-vault-cmk-strategy.md)
- current account / OU model
- intended POC objective

## Day 1: Architecture Boundaries and Control Model

### Theme

Can the target control boundary be made defensible when the Vault OU must remain inside the Production Organization?

### Topics

- Vault OU placement in Prod as a fixed constraint
- isolation from normal production governance
- STDB as the cyber-recovery governance boundary
- service-by-service control boundaries
- identity and access isolation
- detective-control ownership model
- SCP versus IAM versus service-configuration boundaries

### Questions for AWS

1. Given that the Vault OU must remain in the Prod Organization, which Prod-org controls are unavoidable?
2. Which services can be isolated by OU or account, and which are inherently organization-scoped?
3. What is the recommended pattern for GuardDuty, CloudTrail, Config, Security Hub, Access Analyzer, and Detective in this split-governance model?
4. Can STDB own monitoring and detective controls for a Vault OU that physically remains in Prod?
5. What is the recommended identity model for STDB-led administration of Prod-hosted Vault OU resources?
6. Which controls should be implemented with SCPs, which with service configuration, and which with IAM?
7. What compensating controls are mandatory if the Prod Organization is compromised?
8. What CloudTrail, EventBridge, Security Hub, or GuardDuty signals should be treated as mandatory alerts?

### Expected Outputs

By the end of Day 1, the team should have:

- service isolation matrix
- confirmed control boundaries
- detective-control ownership model
- list of hard AWS service constraints
- list of required compensating controls
- open design risks for Day 2 discussion

## Day 2: Backup, KMS, Restore, and Orchestration

### Theme

Can the backup and restore path work safely across Prod and STDB?

### Topics

- AWS Backup logically air-gapped vault behavior
- STDB-owned CMKs in `stdb-key-vault`
- cross-organization KMS key usage
- KMS key policy requirements
- backup copy and restore flows
- AWS RAM / MPA restore behavior
- Step Functions / SSM / EventBridge orchestration
- malware scanning and validation of restored assets

### Questions for AWS

1. Can a Prod-hosted AWS Backup logically air-gapped vault use a CMK owned in the STDB `stdb-key-vault` account?
2. What exact KMS key policy is required for:
   - vault creation
   - copy into the vault
   - restore from the vault
   - RAM sharing
   - MPA restore
3. Which Prod-side principals require KMS usage permissions?
4. Can Prod be restricted to usage-only KMS permissions with no key administration rights?
5. What is the recommended restore orchestration pattern when initiation comes from the STDB Processing OU?
6. Should orchestration use Step Functions, SSM Automation, EventBridge, AWS Backup APIs, or a hybrid pattern?
7. Where should the restore control plane live:
   - Prod Vault OU
   - STDB Processing OU
   - STDB Recovery OU
   - split across accounts
8. Can key access be granted just-in-time during approved restore workflows?
9. What is the recommended MPA implementation pattern?
10. Where should malware scanning and backup validation happen?
11. What AWS-native services are recommended for AMI, EBS, RDS, and restored workload validation?

### Expected Outputs

By the end of Day 2, the team should have:

- validated backup/KMS architecture
- draft KMS key-policy requirements
- restore/copy sequence diagram
- recommended orchestration model
- MPA recommendation
- backup validation pattern
- initial POC scenarios

## Day 3: POC Plan, Operating Model, and Implementation Roadmap

### Theme

How do we prove the design and turn it into implementation work?

### Topics

- POC organizations and account layout
- AWS support for test orgs/accounts
- Terraform / CD model
- deployment model across STDB and Prod Vault OU
- operational runbooks
- monitoring and SIEM integration
- final decisions and open risks

### Questions for AWS

1. Can AWS provide or support creation of test organizations and accounts for the POC?
2. What is the recommended POC account layout?
3. What are the minimum POC success criteria?
4. What failure modes should be explicitly tested?
5. How should continuous deployment be performed across:
   - STDB Organization
   - Prod Vault OU
   - cross-org KMS and AWS Backup policies
6. Should deployment be centralized or split by organization?
7. What is the recommended IaC pattern for this architecture?
8. What should be deployed first:
   - logging
   - delegated admin / security
   - key vault
   - vault POC
   - restore workflow
9. What operational guardrails are required before production rollout?
10. What evidence should be captured from the POC to approve the design?
11. What service quotas, limits, or regional constraints should be checked early?

### Expected Outputs

By the end of Day 3, the team should have:

- POC architecture
- POC success criteria
- implementation backlog
- confirmed CD / IaC model direction
- decision log updates
- open AWS action items
- risk register
- timeline for the next phase

## Additional AWS Asks

Ask AWS to provide or help produce the following artifacts.

### 1. Service Capability Matrix

Requested format:

| Service | Org-scoped? | OU exclusion possible? | Cross-org supported? | Delegated admin options | Recommended pattern |
|---|---|---|---|---|---|

Services to include:

- AWS Backup
- AWS KMS
- AWS RAM
- GuardDuty
- Security Hub
- Detective
- CloudTrail
- AWS Config
- IAM Access Analyzer
- IAM Identity Center
- Step Functions
- EventBridge
- SSM Automation

### 2. Reference Architecture

Ask AWS:

- Can AWS provide a reviewed reference architecture or pattern for cyber recovery using Prod-hosted logically air-gapped vaults and STDB-owned CMKs?

### 3. KMS Key Policy Example

Ask AWS:

- Can AWS provide a sample KMS key policy for an STDB-owned CMK used by a Prod-hosted AWS Backup logically air-gapped vault?

The policy should cover:

- vault creation
- copy into vault
- restore from vault
- AWS Backup service role usage
- RAM / MPA restore path
- usage-only permissions for Prod

### 4. Restore Sequence

Ask AWS:

- Can AWS provide or validate a restore sequence showing principals, accounts, KMS permissions, AWS Backup operations, RAM/MPA, and audit events?

### 5. Threat Model Feedback

Ask AWS:

- If the Prod Organization is compromised, what protections still remain for recovery points in this architecture?
- What risks remain even with STDB-owned CMKs?
- What compensating controls does AWS recommend?

### 6. POC Support

Ask AWS:

- Can AWS help establish or validate two test organizations and participate in POC review?
- If AWS cannot provide test organizations directly, what is the recommended AWS-supported POC setup?

### 7. Known Limitations

Ask AWS:

- What are the known limitations, unsupported patterns, or operational gotchas for this model?

## Suggested Daily Meeting Format

For each day:

1. 15 minutes: recap and decisions needed
2. 60-90 minutes: AWS service deep dive
3. 30 minutes: architecture discussion
4. 30 minutes: decision capture
5. 15 minutes: action items and owners

For longer sessions, add a working block for diagrams, policy review, or POC planning.

## One-Paragraph Workshop Invite

Use the following text for the AWS invite:

```text
We would like to conduct a three-day AWS architecture validation workshop for the STDB/Data Bunker design. The key constraint is that AWS Backup logically air-gapped vaults must reside inside the Production Organization, while STDB is intended to provide the cyber-recovery governance boundary. The workshop objective is to validate the control model, confirm the STDB-owned CMK approach, define the cross-organization restore and monitoring pattern, and agree a POC plan with success criteria and AWS service-specific constraints.
```

## Summary

Recommended structure:

```text
Day 1: Control boundary and detective controls
Day 2: Backup, KMS, restore, orchestration
Day 3: POC, deployment model, roadmap
```

This structure should help AWS bring the right specialists and should help the internal team convert the workshop into decisions and implementation work.
