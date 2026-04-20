1. Cross-Organization CMK Support

Can AWS confirm that a logically air-gapped AWS Backup vault in a Data Bunker account in the Prod AWS Organization can be encrypted at creation time with a symmetric customer-managed KMS key owned by a Key Vault account in a separate STDB AWS Organization?

Please clarify:

Whether this is fully supported for CreateLogicallyAirGappedBackupVault using EncryptionKeyArn.
Whether the key account may be in a different AWS Organization from the vault account.
Whether the key must be in the same Region as the LAG vault.
Whether multi-Region KMS keys are supported or recommended for this pattern.
2. Minimum Key Policy For Vault Creation And Ongoing Use

What is the minimum required KMS key policy on the STDB Key Vault CMK to allow the Prod Org Data Bunker account to create and operate a CMK-encrypted logically air-gapped vault?

Please provide the required principals and actions for:

The Data Bunker role that creates the LAG vault.
The AWS Backup service-linked role or service principal involved in the Data Bunker account.
kms:CreateGrant, kms:DescribeKey, kms:Encrypt, kms:Decrypt, kms:ReEncrypt*, and kms:GenerateDataKey*.
Any required conditions such as kms:ViaService, kms:GrantIsForAWSResource, aws:SourceAccount, or aws:SourceArn.
3. KMS Permissions For Cross-Account Restore From A Shared LAG Vault

When a CMK-encrypted LAG vault is shared by the Data Bunker account to a recovery account in the STDB Org using AWS RAM, which account principals need access to the STDB CMK for restore?

Please clarify:

Whether the recovery account’s AWS Backup service role needs direct access to the CMK.
Whether the Data Bunker account service-linked role still participates during restore.
Whether the destination account service-linked role needs kms:Decrypt, kms:GenerateDataKey*, or kms:CreateGrant.
Whether direct restore from a RAM-shared LAG vault behaves differently from copy-then-restore from a KMS perspective.
4. Key Lifecycle And Recovery Impact

What happens to backup and restore operations if the STDB CMK used by the LAG vault is disabled, scheduled for deletion, deleted, rotated, or has its key policy changed?

Please confirm:

Whether existing recovery points remain visible but unrestorable if the CMK is unavailable.
Whether new copy/backup jobs into the vault fail immediately when the CMK cannot be used.
Whether AWS KMS automatic key rotation is fully transparent for all existing recovery points.
Whether the LAG vault’s encryption key can ever be changed after creation, or whether a new vault is required.
What AWS recommends for break-glass recovery if the CMK policy is accidentally locked down.
5. Recommended Guardrails For STDB-Owned Backup CMKs

What guardrails does AWS recommend around STDB-owned CMKs used for logically air-gapped AWS Backup vaults?

We are especially looking for guidance on:

SCPs or RCPs to prevent unauthorized kms:PutKeyPolicy, kms:DisableKey, and kms:ScheduleKeyDeletion.
Whether MFA or multi-party approval should be required for KMS administrative actions.
Required CloudTrail, EventBridge, or CloudWatch alerts for key policy changes, grant creation, disablement, deletion scheduling, and failed KMS access.
Whether AWS recommends keeping these CMKs in a dedicated Key Vault account separate from the Data Bunker vault account.
Whether AWS still recommends AWS-owned keys over CMKs for ransomware resilience, and how to balance that recommendation against regulatory requirements for customer-managed keys.
