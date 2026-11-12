# DAY 12 — Terraform Drift Detection and Reconciliation

## 1. What is infrastructure drift?

Drift occurs when the real infrastructure changes outside Terraform, so it no longer matches the desired configuration and Terraform's recorded state.

Example: HCL declares an S3 bucket tag `Environment = "dev"`. An operator changes the live AWS tag to `Environment = "prod"` in the console. The next refresh/plan can detect that difference.

Keep three things distinct:
- **Configuration (HCL):** desired settings.
- **State:** Terraform's mapping between resource addresses and remote objects, including recorded attributes.
- **Real infrastructure:** the current object and settings returned by the provider API.

Drift is not automatically a Terraform bug. It may be an emergency manual change, an unauthorized edit, a provider/API normalization, or a change made by another automation system. First identify ownership and intent; then reconcile.

## 2. Small configuration to reproduce drift

Use a disposable AWS account or sandbox. This example manages one S3 bucket and its tags. It assumes the AWS provider is configured by the root module.

~~~hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "app" {
  bucket = "replace-with-a-globally-unique-dev-bucket-name"

  tags = {
    Name        = "app-data"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

output "bucket_arn" {
  description = "ARN of the application bucket"
  value       = aws_s3_bucket.app.arn
}
~~~

Do not use the placeholder bucket name literally. S3 names are globally unique. Keep credentials out of HCL and source control.

## 3. Create, inspect, and record the baseline

Run from the root directory:

~~~bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply
~~~

After apply, inspect the exact address and tracked attributes:

~~~bash
terraform state list
terraform state show aws_s3_bucket.app
terraform show
~~~

Expected resource address:

~~~text
aws_s3_bucket.app
~~~

The address is Terraform's logical identity. The bucket name is the physical identifier. Record both before simulating drift.

## 4. Simulate out-of-band drift

Change a tag directly in AWS without changing the HCL. For example, using AWS CLI:

~~~bash
aws s3api put-bucket-tagging \
  --bucket replace-with-your-bucket-name \
  --tagging 'TagSet=[{Key=Name,Value=app-data},{Key=Environment,Value=prod},{Key=ManagedBy,Value=Terraform}]'
~~~

This changes the live tag from `dev` to `prod`; Terraform configuration still declares `dev`.

Now run:

~~~bash
terraform plan
~~~

Terraform refreshes remote attributes as part of planning and should show a tag difference for `aws_s3_bucket.app`. Exact plan wording and whether a change is in-place can vary by provider version and resource schema. Do not assume every drift is safely repairable in-place.

Inspect the state after the plan:

~~~bash
terraform state show aws_s3_bucket.app
~~~

Modern Terraform planning normally refreshes state information in memory and proposes reconciliation. A normal `terraform plan` does not persist a refreshed state snapshot in the same way as `terraform apply` or the legacy `terraform refresh` command. Avoid using deprecated `terraform refresh` as a routine workflow.

## 5. Understand the plan before reconciling

Classify the plan:
- **In-place update (`~`):** provider can update the existing object without replacing it.
- **Replacement (`-/+`):** Terraform must destroy and recreate the object to achieve the declared configuration. Check why; replacement can cause downtime or data loss.
- **Create (`+`):** Terraform believes a declared object is absent from state/remote reality or needs creation. Verify account, region, backend, workspace, and physical object before applying.
- **No changes:** no difference was detected for the attributes Terraform currently manages. It does not prove every cloud setting is compliant.

Use a saved plan for review:

~~~bash
terraform plan -out=tfplan
terraform show tfplan
~~~

Treat plan files as sensitive artifacts; they can contain sensitive values. Store them securely and use the same reviewed plan for the apply workflow where appropriate.

## 6. Reconcile safely — choose the correct owner

### Case A: Terraform is the source of truth

The manual edit was unauthorized or temporary, and HCL is correct.

1. Confirm the account, region, backend key, workspace, and exact address.
2. Confirm the live object is the expected bucket.
3. Review the plan and verify it changes only the intended attributes.
4. Save and inspect the plan if your workflow requires it.
5. Apply through the approved pipeline or controlled workflow.
6. Run a fresh plan and inspect the live resource to confirm the intended value is restored.

~~~bash
terraform plan -out=tfplan
terraform show tfplan
terraform apply tfplan
terraform plan
terraform state show aws_s3_bucket.app
~~~

### Case B: The manual change is intentional and should remain

Do not apply the old HCL blindly. Update the configuration to represent the approved new value, review the code change, then plan and apply through the normal workflow. This makes the change reproducible and prevents the next run from reverting it.

### Case C: Another system owns the attribute

Define an explicit ownership boundary. For a specific attribute that another controller must manage, a resource's `lifecycle.ignore_changes` can be appropriate, but it suppresses Terraform's reconciliation for those attributes. It is not a general drift fix. Document the owner, scope it narrowly, and ensure security/compliance controls still monitor the value.

Example syntax only — use it only when shared ownership is an intentional design:

~~~hcl
lifecycle {
  ignore_changes = [tags["ExternalSystemStatus"]]
}
~~~

Do not ignore all tags or broad resource attributes merely to make a plan empty.

## 7. Refresh-only plans and state refresh

A refresh-only plan asks Terraform to update its state view to match remote objects without proposing configuration changes to the infrastructure:

~~~bash
terraform plan -refresh-only
~~~

Review what state changes would be recorded. If the out-of-band change is approved and should become the recorded reality, apply the refresh-only plan:

~~~bash
terraform apply -refresh-only
~~~

Important distinction:
- `terraform plan -refresh-only` previews state synchronization.
- `terraform apply -refresh-only` records refreshed remote values in state; it does not change the remote infrastructure to match HCL.
- A normal `terraform plan` compares refreshed remote information with configuration and can propose infrastructure changes.

Use refresh-only only when accepting the remote change into state is intentional. It does not update HCL. If HCL remains different, a later normal plan can still propose reverting the remote change.

## 8. Failure scenario — a production plan unexpectedly replaces a resource

Scenario: a manual change or configuration mismatch causes a plan to show `-/+` for a production resource.

Safe recovery:
1. Stop. Do not apply the plan.
2. Confirm the selected AWS identity with `aws sts get-caller-identity`.
3. Confirm the backend configuration, state key, workspace, region, and provider alias.
4. Inspect the exact address with `terraform state show '<address>'`.
5. Compare the planned before/after values and identify the attribute marked ForceNew/replacement by the provider.
6. Check CloudTrail or the relevant audit trail to identify who changed the live object and when, if available.
7. Decide whether HCL, remote infrastructure, or another controller is authoritative.
8. Correct the source of truth or use a narrowly justified ownership rule; do not remove state or taint a resource as a shortcut.
9. Generate a fresh plan and have the replacement reviewed by the service owner before any apply.

## 9. Useful commands and what they tell you

| Command | Purpose |
|---|---|
| `terraform plan` | Refreshes remote data for planning and shows proposed reconciliation. |
| `terraform plan -refresh-only` | Shows state-only refresh changes. |
| `terraform apply -refresh-only` | Records refreshed remote values in state. |
| `terraform state list` | Lists tracked resource addresses. |
| `terraform state show <address>` | Shows attributes recorded for one address. |
| `terraform show` | Displays the current state snapshot in readable form. |
| `terraform plan -out=tfplan` | Saves a plan for review. |
| `terraform show tfplan` | Inspects a saved plan. |
| `aws sts get-caller-identity` | Confirms the active AWS account and principal. |

Do not use `terraform state rm`, `terraform import`, or `terraform taint` as routine drift-repair commands. They solve different state/ownership problems.

## 10. Common mistakes

- Applying a plan without understanding why the resource changes.
- Assuming a successful plan means the entire account has no unmanaged drift.
- Confusing a refresh-only operation with infrastructure reconciliation.
- Accepting remote values into state while leaving HCL unchanged, then being surprised by the next normal plan.
- Using `ignore_changes` broadly to hide drift.
- Running commands against the wrong account, region, workspace, or backend key.
- Treating every drift event as malicious; verify change history and ownership.
- Deleting state to silence a difference.
- Forgetting that plan files and state may contain sensitive values.
- Assuming every provider attribute is refreshable or every remote change is represented in Terraform state.

## 11. Interview-ready mental model

**Detect → identify the exact address and physical object → determine ownership → inspect the plan → reconcile the source of truth → verify with a fresh plan.**

Terraform detects drift for attributes it reads and manages. A normal plan proposes bringing infrastructure toward HCL; a refresh-only apply accepts remote reality into state without changing infrastructure. The correct action depends on which system is authoritative.

