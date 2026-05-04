# AWS Config — Design Decisions

**Organisation:** AWS Organization — Management Account, Delegated Admin Account, Member Accounts  
**IaC:** Standard Terraform (no Terragrunt)  
**Version:** 1.0 — May 2026

---

## 1. Design Philosophy

Config serves three distinct jobs in this organisation. Each job maps to a specific piece of infrastructure and should be thought about independently.

| Job | Infrastructure | What It Answers |
|---|---|---|
| Record every resource change permanently | Central S3 bucket (Log Archive account) | What did things look like historically? What changed and when? |
| Evaluate resources against compliance rules | Config Rules + Conformance Packs (Delegated Admin) | What is non-compliant right now? |
| Unified inventory view across all accounts | Config Aggregator (Delegated Admin) | What does the org look like right now, across all accounts? |

---

## 2. Account Architecture — What Goes Where

### Who owns what

| Account | Role in Config | Owns | Does NOT Own |
|---|---|---|---|
| Management Account | Bootstrap only | Service access enablement, Delegated admin registration, SCP to protect recorder | Recorder, rules, aggregator, any workloads |
| Delegated Admin Account | Control plane | Org aggregator, Org Config rules, Org conformance packs, Both service-linked roles | Workloads, member account recorders |
| Member Accounts (all) | Data plane | Config recorder (per region), Delivery channel pointing to central S3 | Rule authorship, aggregator, conformance pack authorship |
| Log Archive Account | Audit archive | Central S3 Config delivery bucket | Recorder, rules, aggregator |

### Why this separation

| Decision | Rationale |
|---|---|
| Management account is minimal | AWS best practice — management account should only be used for tasks that require it. Reduces blast radius if management credentials are compromised. |
| Delegated admin is a dedicated governance account | Keeps Config control plane separate from workload accounts. Same account used for Security Hub and GuardDuty centralisation. |
| S3 bucket in Log Archive, not Delegated Admin | Puts the audit trail in a separate trust boundary. A compromised delegated admin account cannot tamper with the historical record. |
| Member accounts receive rules, don't author them | Ensures consistent compliance policy across the org. Teams cannot disable or modify rules in their own accounts. |

---

## 3. Prerequisites

### Must be true before any Terraform runs

| Prerequisite | Where | Notes |
|---|---|---|
| AWS Organizations All Features enabled | Management Account | Consolidated Billing mode is not enough — trusted service access requires All Features |
| `TerraformDeploymentRole` IAM role | Every account | Bootstrapped manually or via account vending before Config Terraform runs |
| Remote state S3 bucket + DynamoDB lock table | Every account | One per account — isolated state, no shared state bucket |
| Central S3 Config delivery bucket created | Log Archive Account | Must exist with correct bucket policy before recorders start. Policy uses `aws:SourceOrgID` to cover all current and future accounts automatically |
| Two service-linked roles created manually | Delegated Admin Account | `config.amazonaws.com` and `config-multiaccountsetup.amazonaws.com`. Unlike member accounts, these are NOT auto-created for delegated admins — most commonly missed step |

---

## 4. The Central S3 Bucket

### What it is for

| Question Type | Can S3 Answer It? | Example |
|---|---|---|
| What did a resource look like 6 months ago? | ✅ Yes | Reconstruct IAM role configuration at a point in time |
| Prove encryption was enabled throughout last year | ✅ Yes | Compliance evidence for auditors and regulators |
| What changed on this resource and when? | ✅ Yes | Full change history per resource, back to when recording started |
| Run SQL queries across years of history | ✅ Yes (via Athena) | "Every IAM role that ever had AdministratorAccess attached" |
| What does the org look like right now? | ❌ No | Use the Aggregator for current-state questions |
| Compliance dashboard and rule scores | ❌ No | Use the Aggregator for this |

### Key design decisions

| Decision | Choice | Reason |
|---|---|---|
| Bucket location | Log Archive account | Separate trust boundary from all operational accounts |
| Encryption | SSE-KMS with dedicated key | Key access is independently auditable and revocable |
| Versioning | Enabled | Protects against accidental overwrite |
| Object Lock | Compliance mode | Prevents deletion even by bucket owner — required for regulatory frameworks |
| Bucket policy scope | `aws:SourceOrgID` condition | Covers all current and future accounts without policy updates on new account onboarding |
| MFA Delete | Enabled | Additional protection against deletion — requires root credentials, set manually |

---

## 5. The Aggregator

### What it is for

| Question Type | Can the Aggregator Answer It? | Example |
|---|---|---|
| Which S3 buckets are publicly accessible right now? | ✅ Yes | Advanced Query across all accounts in seconds |
| Which EC2 instances are not enforcing IMDSv2? | ✅ Yes | Real-time inventory query |
| What is the org-wide compliance score for CIS? | ✅ Yes | Compliance dashboard, per-rule and per-account breakdown |
| Which accounts have CloudTrail currently disabled? | ✅ Yes | Cross-account rule compliance status |
| What did a resource look like last month? | ❌ No | Aggregator only holds current state — use S3 for history |
| Prove controls were in place throughout last year | ❌ No | Use S3 + Athena for historical evidence |

### Key design decisions

| Decision | Choice | Reason |
|---|---|---|
| Aggregator type | Organisation aggregator | Automatically includes all current and future member accounts. No per-account authorization needed. |
| Region scope | All regions | No blind spots — if a region is active anywhere in the org, it is aggregated |
| Cost | Free | Aggregation itself has no additional charge. Only source account recordings are billed. |
| Location | Delegated Admin account | Co-located with org rules and conformance packs for a single operational account |

---

## 6. Aggregator vs S3 — When to Use Which

| Scenario | Use | Tool |
|---|---|---|
| Daily compliance monitoring | Current state | Aggregator — Advanced Query |
| Incident response — what changed right before the alert? | History | S3 + Athena |
| Weekly compliance dashboard for leadership | Current state | Aggregator — compliance scores |
| Annual audit — prove controls were in place all year | History | S3 + Athena |
| Find all unencrypted EBS volumes across the org | Current state | Aggregator — Advanced Query |
| Track how a specific account's posture changed over time | History | S3 + Athena |
| New account check — is the recorder running? | Current state | Aggregator |
| Regulatory evidence package | History | S3 |

---

## 7. Config Rules — Design Decisions

### Governing principles

| Principle | Detail |
|---|---|
| Start minimal | Six rules only at launch. Conformance packs come later, after the environment is understood. |
| Monitor before remediate | All rules run monitor-only for the first 4 weeks. No automated actions until findings are triaged. |
| Wave-based expansion | Rules are added in thematic batches. Next wave starts only when the current wave's findings are resolved or formally excepted. |
| Rules are version-controlled | The rule set lives in a Terraform map. Every change is a pull request — auditable, reviewable, reversible. |
| Org rules only | All rules deployed as org-level rules from the delegated admin. No account-level rules in the baseline. Member accounts cannot modify them. |

---

## 8. Starting Rule Set — Minimum Viable Compliance

Six rules deployed at launch. These were chosen because they have near-zero false positives, protect against the most critical risks, and represent genuine problems if non-compliant.

| Rule | AWS Identifier | Why It Is in the Minimum Set |
|---|---|---|
| CloudTrail enabled | `CLOUD_TRAIL_ENABLED` | Without this, there is no audit log. Security investigations are impossible. |
| Config recorder enabled | `CLOUD_TRAIL_ENABLED` (self-monitoring) | Detects if the recorder is stopped in any account — Config monitoring itself. |
| Root account MFA enabled | `ROOT_ACCOUNT_MFA_ENABLED` | Root without MFA is an existential account risk. Non-negotiable. |
| Root has no access keys | `IAM_ROOT_ACCESS_KEY_CHECK` | Root access keys should never exist. Zero expected false positives. |
| No S3 public read | `S3_BUCKET_PUBLIC_READ_PROHIBITED` | Publicly readable S3 is the source of the most common accidental data exposures. |
| No S3 public write | `S3_BUCKET_PUBLIC_WRITE_PROHIBITED` | Publicly writable S3 enables data injection and storage abuse. |

No conformance packs at this stage. The reasoning: a full CIS pack deployed into a new environment will surface 50–100+ findings immediately, producing a dashboard that is permanently red and which teams stop looking at. Individual targeted rules first.

---

## 9. Wave-Based Rule Rollout

Move to the next wave only when the current wave's findings are resolved, formally accepted as exceptions, or suppressed with documented justification.

| Wave | Theme | Rules Added | Watch Out For |
|---|---|---|---|
| Wave 1 (Launch) | Critical baseline | CloudTrail, Config self-monitor, Root MFA, Root access keys, S3 public access (read + write) | Legacy accounts with CloudTrail gaps |
| Wave 2 | IAM hygiene | Access key rotation (90 days), IAM password policy, MFA for all console users | Legacy IAM users, service accounts with old keys |
| Wave 3 | Encryption at rest | EBS volumes encrypted, RDS instances encrypted, S3 default encryption enabled | Older resources that predate the encryption policy — need migration plan, not just a finding |
| Wave 4 | Network baseline | Default security group closed, SSH restricted, common management ports restricted | Dev environments often have intentional exceptions — scope to production OUs first |
| Wave 5 | Conformance pack | CIS AWS Foundations Benchmark (org conformance pack) | By Wave 5 the environment should be ~80–90% compliant, making the initial score meaningful |
| Wave 6+ | Regulatory specific | PCI DSS, HIPAA, NIST 800-53 conformance packs as required | Only add if the organisation has a specific regulatory obligation for that framework |

---

## 10. Conformance Packs — When and How

| Question | Answer |
|---|---|
| When do we deploy the first conformance pack? | Wave 5 — after individual rule waves are clean |
| Which pack first? | CIS AWS Foundations Benchmark — comprehensive, widely recognised, maps to most other frameworks |
| Why not deploy a pack on day one? | A pack deployed cold into a new environment produces a 30–40% compliance score with hundreds of findings. That score is not useful operationally or for reporting. Starting with individual rules brings the baseline to 80–90% first. |
| What does a conformance pack add over individual rules? | A single framework-aligned compliance score that can be tracked over time and shown to auditors. Individual rules drive operational remediation; the conformance pack score drives the governance narrative. |
| Can packs and individual rules overlap? | Yes, but avoid it. If the same rule exists both standalone and inside a pack, Config bills it as two separate evaluations and reports two separate results. |
| How are custom org policies encoded? | Custom conformance packs using a mix of managed rules and custom CloudFormation Guard policy rules. Introduced only after the team has sufficient Config operational experience. |

---

## 11. Remediation Approach

### Maturity stages

| Stage | Mode | Config | When to Move to Next Stage |
|---|---|---|---|
| Stage 1 | Monitor only | No remediation configured | After 4 weeks, findings triaged, team understands the blast radius of each rule |
| Stage 2 | Manual remediation | SSM Automation runbook available, triggered by a human from the Config console | When a finding type is well understood and the fix is unambiguous |
| Stage 3 | Automatic remediation | Config auto-triggers SSM runbook on non-compliance | Only for findings where the action is unambiguous and low-risk (S3 public access, EBS encryption default) |
| Stage 4 | Approved auto-remediation | High-risk remediations route through SSM Change Manager — human approval required before execution | When remediations touch IAM, security groups, or KMS |

### What gets auto-remediated vs what requires approval

| Finding Type | Remediation Mode | Reason |
|---|---|---|
| S3 bucket public access enabled | Automatic | Unambiguous risk, reverting is non-destructive |
| EBS encryption-by-default disabled | Automatic | Enabling it does not affect existing volumes, safe to automate |
| IAM access key not rotated | Approved (Change Manager) | Rotating a key can break a running workload |
| Security group allows unrestricted SSH | Approved (Change Manager) | Removing access can lock out legitimate users |
| KMS key rotation disabled | Approved (Change Manager) | Key management changes require careful review |
| Root MFA not enabled | Manual only | Cannot be automated — requires console action by a human with root credentials |

---

## 12. Ongoing Operations

### How to know when to add more rules

| Signal | Action |
|---|---|
| Current wave showing near-100% compliance org-wide | Ready to move to next wave |
| Findings from current wave are unresolved or unreviewed | Stay in current wave — do not add more |
| High-volume findings that nobody is acting on | Scope the rule more narrowly, or move to a later wave |
| Zero findings from a rule for 90+ days | Review whether the rule still adds value — candidate for removal |

### Exception and suppression management

| Type | Handling |
|---|---|
| Resource is intentionally non-compliant for valid business reason | Config resource-level exception, documented in Terraform — version controlled and visible in code review |
| Entire account has a different compliance requirement | Exclude the account from the specific org rule or conformance pack via `excluded_accounts` in Terraform |
| Rule produces false positives in a specific environment | Scope the rule more narrowly using resource tags or account targeting before suppressing |

### New account onboarding

| Step | Detail |
|---|---|
| Run bootstrap script | Creates account root module from template, replaces account ID placeholder, updates `accounts.json` |
| CI/CD pipeline applies recorder | `deploy_all_accounts.sh` runs `terraform apply` for the new account directory |
| Org rules auto-apply | AWS Config automatically pushes org rules and conformance packs to the new account once recorder is confirmed running |
| 7-hour window | If the recorder is not running within 7 hours of the account joining the org, org rule deployment must be manually re-triggered from the delegated admin account |

---

## 13. Summary of All Key Decisions

| Area | Decision | Rationale |
|---|---|---|
| Management account role | Bootstrap only — no recorder or rules | Keeps management account minimal; reduces attack surface |
| Delegated admin | Dedicated governance account | Separates Config control plane from workloads; single operational account |
| Recorder scope | All accounts, all active regions | Full visibility — any unmonitored region is a blind spot |
| Global resource types | Recorded in us-east-1 only per account | Prevents duplicate recording of IAM and other global resources; reduces cost |
| Recording frequency | Continuous default, Daily override for high-volume compute | Real-time security visibility balanced against cost |
| S3 bucket location | Log Archive account | Separate trust boundary; tamper-proof audit trail |
| S3 bucket protection | Object Lock (Compliance mode) + SSE-KMS + MFA Delete | Meets regulatory evidence requirements; prevents deletion |
| S3 bucket policy | `aws:SourceOrgID` condition | Covers new accounts automatically without policy updates |
| Aggregator type | Org aggregator, all regions | Auto-includes new accounts; no per-account authorization |
| Rule deployment | Org-level from delegated admin | Consistent policy across all accounts; member accounts cannot override |
| Starting rule set | 6 rules only | Avoids alert fatigue; establishes baseline before expanding |
| Rollout approach | Wave-based, move only when current wave is clean | Prevents permanent red dashboards that teams stop using |
| Conformance packs | Introduced at Wave 5 | Meaningful compliance score requires a reasonably clean baseline |
| Remediation | Monitor-only first; auto only for unambiguous low-risk | Prevents accidental disruption; human approval for IAM and network |
| Terraform structure | Per-account root modules, shared reusable modules | No external tooling dependency; provider config separated from resource logic |
| Suppression/exceptions | Managed in Terraform | Version-controlled, reviewable, auditable |
