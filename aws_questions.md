Based on the AWS blog and the STDB workspace design, these are the 5 questions I’d take to AWS. They are framed to force implementation-level answers, not just “yes, supported.”

Source blog: [Encrypt AWS Backup logically air-gapped vaults with customer-managed keys](https://aws.amazon.com/blogs/storage/encrypt-aws-backup-logically-air-gapped-vaults-with-customer-managed-keys/)

**Context To Give AWS**

STDB’s target pattern is:

- Workload accounts remain in the Prod AWS Organization.
- Isolated Data Bunker vault accounts also remain in the Prod Org because AWS Backup cross-account copy requires source and destination accounts to be in the same AWS Organization.
- The KMS keys for the Data Bunker logically air-gapped vaults are owned by a Key Vault account in the separate STDB/DataBunker Organization.
- STDB wants Data Bunker-controlled “pull” orchestration, not workload-owned backup-plan `copy_action` pushes.
- Recovery accounts are created just in time in the STDB Org and receive account-scoped AWS RAM sharing only during restore events.

Workspace context used: [stdb-backup-lag-hld.md](/Users/krishnansubramanian/Library/CloudStorage/OneDrive-Personal/Documents/STDB_2026/stdb-backup-lag-hld.md), [stdb-backup-lag-design.md](/Users/krishnansubramanian/Library/CloudStorage/OneDrive-Personal/Documents/STDB_2026/stdb-backup-lag-design.md), [lld-01-account-boundary-scp-envelope.md](/Users/krishnansubramanian/Library/CloudStorage/OneDrive-Personal/Documents/STDB_2026/lld-01-account-boundary-scp-envelope.md).

**1. Cross-Organization CMK Support For LAG Vault Creation**

Can AWS confirm that a logically air-gapped backup vault in a Data Bunker account in the Prod Org can be encrypted at creation time with a symmetric customer-managed KMS key owned by a Key Vault account in a separate STDB AWS Organization?

Follow-ups we need answered:

- Is cross-Organization CMK usage fully supported for `CreateLogicallyAirGappedBackupVault --encryption-key-arn`, or only cross-account within the same Organization?
- Are there any hidden prerequisites around AWS Organizations trusted access, RAM, KMS grants, or service-linked roles?
- Is one regional CMK per Region required, or can any multi-Region KMS key pattern be used with AWS Backup LAG vaults?

**Why this matters:** This is the central design assumption. The AWS blog says the designated key vault account can be in the same AWS Organization or a different Organization, but we need AWS to validate the exact STDB account topology.

**2. Required KMS Key Policy For Create, Copy, Share, And Restore**

What exact KMS key policy statements are required on the STDB Key Vault CMK for the full lifecycle: LAG vault creation, copy into the vault, RAM-based recovery-account restore, and any re-copy/export workflow?

Please provide the minimum required principals and actions for:

- The Data Bunker account role that creates the LAG vault.
- The AWS Backup service path in the Data Bunker account.
- The workload account role that initiates `StartCopyJob` from the source-account context.
- The STDB recovery account role used during RAM-based restore.
- The destination service-linked role, if restore creates resources in the recovery account.

**Why this matters:** The blog gives example policy shapes, but STDB needs a production-ready trust model. The current design separates workload CMKs from Data Bunker LAG vault CMKs, so over-broad key policy guidance would weaken the boundary.

**3. Pull-Orchestration Compatibility With AWS Backup APIs**

Can AWS confirm the supported API model for STDB’s “pull” pattern, where Data Bunker-owned orchestration assumes a narrow role in each workload account and starts the copy job from the workload account context into the Data Bunker LAG vault?

Specific points to validate:

- Is `StartCopyJob` still required to be called from the source/workload account for cross-account copies into a LAG vault?
- Can the destination be a logically air-gapped vault encrypted with a CMK from a third account in a separate Organization?
- What IAM, backup vault access policy, and KMS permissions are required on the workload source vault and workload CMK?
- Are there service-specific constraints for EFS recovery points copied this way?

**Why this matters:** STDB does not want workload accounts to own scheduled copy actions into the bunker. The design depends on Data Bunker owning the decision while satisfying AWS Backup’s source-account API requirement.

**4. Recovery Sharing Across Organizations With CMK-Encrypted LAG Vaults**

For a CMK-encrypted logically air-gapped vault shared through AWS RAM from the Data Bunker account in the Prod Org to a just-in-time recovery account in the STDB Org, what exact permissions and workflow are required for restore?

Please clarify:

- Whether the LAG vault can be shared directly to an individual account ID in another Organization.
- Whether the recovery account must be granted KMS access before the RAM share is accepted, before restore starts, or only at restore time.
- Which principal decrypts or uses the CMK during direct restore from the shared LAG vault.
- Whether removing the RAM share after restore affects already-restored resources or only future access to recovery points.

**Why this matters:** STDB cannot share to an STDB OU because RAM OU principals are limited to the sharing account’s Organization. The solution needs an account-scoped, just-in-time restore pattern that AWS confirms as supported.

**5. Guardrails And Failure Modes For CMK Loss, Disablement, Rotation, Or Policy Change**

What are the exact operational consequences if the STDB-owned CMK used by a LAG vault is disabled, scheduled for deletion, rotated, or has its key policy accidentally changed?

We need AWS guidance on:

- Whether existing recovery points remain listed but unrestorable if the CMK becomes unavailable.
- Whether AWS Backup copy jobs into the LAG vault fail immediately if the CMK policy/grants break.
- Whether automatic KMS rotation is transparent for all existing and future recovery points.
- What CloudTrail/EventBridge signals AWS recommends for detecting key-policy drift, grant failures, disabled keys, and failed restore attempts.
- Whether AWS recommends AOK over CMK for ransomware resilience even when regulatory requirements prefer CMK control.

**Why this matters:** The blog notes AOK remains AWS’s recommended approach for most use cases, while CMKs support stricter governance. STDB needs to understand the security tradeoff: CMK control helps compliance, but key unavailability could become a recovery dependency.
