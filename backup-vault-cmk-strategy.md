# Backup Vault CMK Strategy

## Purpose

This document defines the initial CMK ownership strategy for AWS Backup logically air-gapped vaults used by the STDB / Data Bunker design.

The key design decision is:

- the air-gapped backup vault lives in the Prod Organization Vault OU
- the customer-managed KMS key lives in the STDB Organization

This is intended to separate the backup storage plane from the encryption key control plane.

## Design Decision

The selected approach is:

```text
Prod Organization / Vault OU
  owns the AWS Backup logically air-gapped vault

STDB Organization / stdb-key-vault
  owns the customer-managed KMS keys
  owns key lifecycle, policy, audit, and monitoring

Prod Vault OU account
  receives only the KMS usage permissions required by AWS Backup
```

This decision is tracked as:

- `DL-012`: CMK ownership for Prod-hosted logically air-gapped vaults

Status:

- `In Review` until POC validation is complete

## Account Model

Add a new account to the STDB organization:

```text
stdb-key-vault
```

Recommended placement:

```text
Security OU
```

Purpose:

- own CMKs used by Prod-hosted logically air-gapped backup vaults
- control key policy and lifecycle
- monitor KMS usage and administration events
- provide a stronger cyber-recovery boundary even though the vault itself must remain in Prod

## Key Model

Initial key model:

```text
one CMK per backup vault, per Region as required
```

Example aliases:

```text
alias/stdb/prod-airgapped-vault/<vault-name>/ap-south-1
alias/stdb/prod-airgapped-vault/<vault-name>/ap-south-2
```

KMS keys are Regional. If an air-gapped vault exists in more than one Region, create the required CMK in each Region.

## Region Model

The `stdb-key-vault` account should enable all approved backup/recovery Regions required for vault encryption.

For the current STDB baseline, this is expected to include:

- `ap-south-1`
- `ap-south-2`

Avoid enabling every AWS Region unless the approved recovery-region model requires it.

## Key Administration Boundary

CMK administration should stay in STDB.

Allowed administrators:

- STDB platform/security administration roles
- approved STDB automation roles
- approved break-glass path

Not allowed as key administrators:

- Prod organization administrators
- Prod Vault OU administrators
- workload account roles
- normal backup operator roles

Prod should receive usage-only permissions, not lifecycle or policy permissions.

## Prod Usage Permissions

The Prod Vault OU account should be granted only the KMS permissions required for AWS Backup vault operation.

The exact principal list must be validated in POC and with AWS, but likely candidates include:

- the role that creates the AWS Backup logically air-gapped vault
- AWS Backup service role or service-linked role in the Prod Vault OU account
- roles involved in backup copy and restore workflows

The policy should avoid granting Prod permissions such as:

- `kms:PutKeyPolicy`
- `kms:DisableKey`
- `kms:ScheduleKeyDeletion`
- `kms:CancelKeyDeletion`
- broad `kms:*`

Usage permissions should be constrained where supported, for example:

- `kms:ViaService = backup.<region>.amazonaws.com`
- `kms:GrantIsForAWSResource = true`
- explicit principal ARNs
- explicit account boundaries

## Monitoring and Audit

STDB owns monitoring and audit for these CMKs.

Monitor at minimum:

- `PutKeyPolicy`
- `DisableKey`
- `ScheduleKeyDeletion`
- `CancelKeyDeletion`
- `CreateGrant`
- `RevokeGrant`
- unexpected `Decrypt`
- unexpected principal usage

Audit evidence should be retained through the STDB logging and security monitoring model.

The final monitoring destination will depend on the STDB SIEM/detective-control decision.

## POC Scope

A POC is required before this design is finalized.

The POC should validate:

1. `stdb-key-vault` can create a CMK in the approved Region.
2. Prod Vault OU account can create a logically air-gapped vault using the STDB-owned CMK ARN.
3. AWS Backup can copy recovery points into the Prod-hosted vault using the STDB-owned CMK.
4. Restore or sharing workflows work with the STDB-owned CMK.
5. STDB receives usable KMS and CloudTrail audit evidence.
6. Prod has usage-only key access and cannot administer, disable, delete, or repolicy the key.

## Open AWS Validation Questions

Ask AWS to confirm:

- Is cross-organization CMK use fully supported for AWS Backup logically air-gapped vault encryption?
- What exact KMS key policy is required for vault creation, copy, restore, RAM sharing, and MPA workflows?
- Which Prod-side principals need CMK usage permissions?
- Can the Prod Vault OU account create and operate the vault without broader trust into STDB?
- Are there limitations with one CMK per vault?
- Can key access be granted just-in-time for approved restore workflows?
- What KMS and CloudTrail events should be monitored as mandatory detective controls?

## Implementation Notes

Do not create production-grade air-gapped vaults until the CMK strategy is confirmed.

The encryption key for a logically air-gapped backup vault is a foundational design choice. If the key cannot be changed after vault creation, the key ownership, policy, and monitoring model must be validated before real vaults are created.

## Summary

The STDB design will use:

```text
Prod-hosted air-gapped vaults
STDB-owned CMKs
usage-only permissions for Prod
STDB-owned lifecycle, audit, and monitoring
```

This gives the design a stronger cyber-recovery control boundary while respecting the AWS Backup requirement that the logically air-gapped vault remains inside the Prod organization.
