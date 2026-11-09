# DAY 9 — Remote State with S3 Backend and State Locking

## 1. Why remote state?

Terraform state maps Terraform resource addresses to real infrastructure objects and stores attributes Terraform needs for future plans.

With local state, the state file lives in the working directory. That is risky for a team: developers and CI jobs can use different copies, local disks can be lost, and concurrent operations can conflict.

An S3 backend gives a team a shared state location. State is not just a cache; it can contain sensitive values, so the backend must be protected like production data.

Core responsibilities:
- **S3 backend:** durable remote storage for the state object.
- **Locking:** coordinates writers so two Terraform operations do not update the same state concurrently.
- **Versioning:** provides recoverable previous object versions; it is not a substitute for locking.
- **IAM and bucket policy:** determine who can list, read, write, and lock the state.
- **Encryption and public-access controls:** protect stored data and prevent unintended public exposure.

## 2. Recommended architecture

Use a dedicated state bucket, ideally in a dedicated tooling/security account. Separate state keys by project, environment, and region where appropriate.

Example:

```text
s3://company-terraform-state/
  platform/dev/ap-south-1/terraform.tfstate
  platform/prod/ap-south-1/terraform.tfstate
  networking/shared/ap-south-1/terraform.tfstate
```

Use separate Terraform root configurations and state keys for environments that need independent lifecycle, permissions, or blast radius. Do not put every environment into one shared state just because S3 can store it.

## 3. Bootstrap the backend separately

The S3 bucket must exist before Terraform can initialize an S3 backend. This creates a bootstrap problem: how do you manage the backend bucket itself?

A common pattern is a small, separately managed bootstrap configuration:
1. Create the state bucket and its security controls using a dedicated bootstrap root module.
2. Protect that bootstrap state separately (for example, a tightly controlled local state during initial setup, then migrate it to a separately provisioned backend).
3. Restrict who can modify or delete the bucket and its policy.
4. Configure application/root modules to use the bucket as their backend.

Avoid a circular design where a root module needs the bucket to initialize its backend but also tries to create that same bucket as one of its resources.

## 4. Configure the S3 backend

Backend configuration belongs in the root module's `terraform` block, not in a provider block.

```hcl
terraform {
  required_version = ">= 1.10.0"

  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "platform/dev/ap-south-1/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Important:
- `bucket`: existing S3 bucket name.
- `key`: object key for this root module's state.
- `region`: region where the backend bucket lives (not necessarily the region where managed resources live).
- `encrypt = true`: requests server-side encryption for the state object.
- `use_lockfile = true`: enables the S3 backend's native lockfile mechanism. S3 lockfiles require Terraform 1.10 or later.

The lock object is created beside the state object with the `.tflock` suffix. For the example above:

```text
platform/dev/ap-south-1/terraform.tfstate
platform/dev/ap-south-1/terraform.tfstate.tflock
```

The S3 backend also supports DynamoDB-based locking in older configurations, but DynamoDB locking is deprecated. Prefer native S3 lockfiles for new configurations unless an established platform standard requires otherwise.

Backend blocks are initialized before normal input-variable evaluation. Do not try to use `var.*`, locals, data sources, or resource references in backend configuration. Use static values or partial backend configuration supplied during initialization.

## 5. Keep backend configuration environment-specific

For example, use a small backend config file:

```hcl
bucket       = "company-terraform-state"
key          = "platform/dev/ap-south-1/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

Initialize with:

```bash
terraform init -backend-config=dev.backend.hcl
```

A production backend config would use a different key, and may use a different bucket/account if your platform architecture requires it.

Do not commit credentials or secrets in backend config files. Backend configuration can be recorded in the local `.terraform` directory and plan-related metadata; treat it as configuration that may contain sensitive endpoint or credential details. Use an approved credential chain or workload identity.

## 6. Secure the S3 bucket

The backend bucket should have these controls:

### A. Block public access

Enable all four S3 Block Public Access settings at the bucket level (and preferably account level):
- Block public ACLs.
- Ignore public ACLs.
- Block public bucket policies.
- Restrict public bucket policies.

Do not use public ACLs or public bucket policies for Terraform state.

### B. Enable versioning

Enable S3 versioning so an accidental overwrite or deletion can be recovered from a prior object version. Restrict who can permanently delete object versions.

Versioning preserves history; it does not prevent concurrent writes and does not replace state locking.

### C. Enforce encryption

Use server-side encryption. SSE-S3 is a simple baseline; SSE-KMS is appropriate when you need customer-managed key control, key policies, audit requirements, or separation of duties.

For SSE-KMS, ensure the Terraform execution identity and S3 service path have the required KMS permissions, commonly including `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`, and `kms:DescribeKey`, subject to the exact key policy and integration. A correct S3 policy alone does not guarantee KMS access.

### D. Require TLS

A bucket policy can explicitly deny requests where `aws:SecureTransport` is false. This is a defensive guardrail; ensure the policy is reviewed so it does not block legitimate AWS service access unexpectedly.

### E. Restrict access to exact prefixes

Give each deployment role access only to the state key and lock key it needs, plus the bucket-level listing permission scoped to the expected prefix. Avoid broad `s3:*` or bucket-wide write permissions.

## 7. Understand the IAM permissions

For a single state key, the execution role typically needs:

- `s3:ListBucket` on the bucket, constrained by the state prefix.
- `s3:GetObject` and `s3:PutObject` on the state object.
- With S3 lockfile enabled, `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` on the lock object.

Example policy shape (replace bucket, prefix, and role context with approved values):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyThisStatePrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-terraform-state",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "platform/dev/ap-south-1/terraform.tfstate",
            "platform/dev/ap-south-1/terraform.tfstate.tflock"
          ]
        }
      }
    },
    {
      "Sid": "ReadWriteStateObject",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::company-terraform-state/platform/dev/ap-south-1/terraform.tfstate"
    },
    {
      "Sid": "ManageStateLock",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::company-terraform-state/platform/dev/ap-south-1/terraform.tfstate.tflock"
    }
  ]
}
```

This is an illustrative least-privilege policy shape, not a complete universal policy. Terraform operations, workspace use, bucket policy conditions, encryption mode, and KMS key policy can change the required permissions. Validate the actual API calls and prefix behavior for your backend design.

Remember the policy evaluation rules:
- An explicit deny overrides an allow.
- Identity-based and resource-based policies both matter.
- Permission boundaries, session policies, SCPs, VPC endpoint policies, and KMS key policies can also restrict access.
- `s3:ListBucket` applies to the bucket ARN; object actions apply to object ARNs.

## 8. Test the S3 backend operation and identify the denied statement

Use a disposable state bucket and a test execution role. Do not deliberately break a production backend policy.

A practical test sequence:

```bash
aws sts get-caller-identity
aws s3api get-bucket-location --bucket company-terraform-state
aws s3api get-public-access-block --bucket company-terraform-state
aws s3api get-bucket-versioning --bucket company-terraform-state
aws s3api get-bucket-encryption --bucket company-terraform-state

terraform init -reconfigure
terraform plan
```

If Terraform returns `AccessDenied`, classify the failing operation:
- Failure while listing the bucket: inspect `s3:ListBucket` on the bucket ARN and the `s3:prefix` condition.
- Failure reading the state: inspect `s3:GetObject` on the exact state object ARN.
- Failure writing state: inspect `s3:PutObject` on the exact state object ARN.
- Failure acquiring/releasing a lock: inspect `GetObject`, `PutObject`, and `DeleteObject` on the exact `.tflock` object ARN.
- Failure involving SSE-KMS: inspect the KMS key policy and caller permissions as well as S3 permissions.

Use CloudTrail S3 data events if they are enabled for the bucket to identify the denied API operation and principal. S3 object-level data events are not necessarily enabled by default and can incur logging costs. For policy reasoning, inspect the role's attached/inline policies, bucket policy, permission boundary, SCPs, endpoint policy, and KMS key policy. IAM policy simulation can help test identity policies but does not fully replace evaluation of every resource policy and organization control.

Do not guess the denied statement from the generic error alone. Match the denied action, principal, resource ARN, and request context to the relevant policy statement.

## 9. State locking behavior

When Terraform begins an operation that needs to modify state, it attempts to acquire a lock. With `use_lockfile = true`, the S3 backend creates the lock object.

If another operation already holds the lock, the new operation waits or fails with a lock error rather than proceeding as a concurrent writer.

Useful commands:

```bash
terraform plan
terraform apply
terraform force-unlock LOCK_ID
```

Use `force-unlock` only after confirming the lock belongs to an operation that is no longer running and after following team incident procedures. A lock is a safety mechanism, not an obstacle to bypass. Never force-unlock just because a pipeline is slow.

## 10. Migrate existing local state to S3 safely

After the bucket and its security controls exist, add the S3 backend configuration to the root module.

Then run:

```bash
terraform init -migrate-state
```

Terraform prompts to copy the existing state to the new backend. Review the prompt and confirm only when the target bucket/key is correct and access is verified.

Before migration:
1. Stop concurrent local and CI Terraform operations.
2. Back up the existing state securely.
3. Verify the destination bucket, key, encryption, versioning, IAM, and lockfile access.
4. Run `terraform init -migrate-state`.
5. Confirm the migration completed.
6. Run `terraform state list` and `terraform plan`.
7. Confirm the plan does not propose recreating existing infrastructure.

Use `terraform init -reconfigure` when you want to initialize the working directory against a different backend configuration without asking Terraform to migrate the previous state. Understand the consequences before choosing between `-migrate-state` and `-reconfigure`.

## 11. Failure scenario — Terraform cannot read or lock state

Scenario: a CI pipeline fails during `terraform init` or `terraform plan` with an S3 `AccessDenied` error.

Safe troubleshooting:
1. Stop before apply; do not switch to local state as an unreviewed workaround.
2. Run `aws sts get-caller-identity` in the same job context to identify the actual principal.
3. Confirm the configured bucket, region, key, and workspace prefix.
4. Determine whether the failure is bucket listing, state object read/write, lock object access, or KMS.
5. Match the failed action and resource ARN to the exact IAM or bucket-policy statement.
6. Check for explicit denies, permission boundaries, SCPs, endpoint policies, and KMS key policy.
7. Correct only the missing permission or conflicting condition.
8. Retry `terraform init` or `terraform plan` and confirm the state and lock operations succeed.
9. Review the plan before any apply.

Never grant `s3:*` on `*` simply to make the pipeline green.

## 12. Common mistakes

- Creating the S3 backend bucket inside the same root module that needs it to initialize.
- Forgetting `use_lockfile = true` and assuming S3 object versioning is locking.
- Using an old DynamoDB locking example for a new backend without understanding its deprecation.
- Granting object permissions but forgetting `s3:ListBucket` on the bucket ARN.
- Granting `ListBucket` but scoping its prefix condition incorrectly.
- Forgetting permissions on the `.tflock` object.
- Assuming S3 permissions are sufficient when SSE-KMS is used.
- Treating `encrypt = true` as a replacement for IAM restrictions and bucket security controls.
- Leaving public access controls disabled.
- Forgetting versioning or allowing broad permanent deletion of versions.
- Running `force-unlock` while another operation is active.
- Using `-reconfigure` when the intention was to migrate existing state.
- Sharing one state key across independent environments or teams.

## 13. Interview-ready mental model

**Backend configuration locates state → IAM and bucket policy authorize the request → encryption protects the object → lockfile coordinates writers → versioning supports recovery → plan verifies infrastructure alignment.**

When S3 backend access fails, identify the exact operation first: list, read, write, lock, unlock, or KMS. Then match the action and resource ARN to the policy statement that allowed or denied it.
