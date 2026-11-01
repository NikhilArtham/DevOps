# Terraform Labs

## DAY 1 — Professional Workflow, Providers and Versions

### Lab 1: Build a small Terraform root module

Create `versions.tf`, `providers.tf`, `variables.tf`, and `main.tf` using the AWS provider.

Tasks:

1. Add a Terraform CLI version constraint.
2. Add an AWS provider source and version constraint.
3. Configure the AWS region through a variable.
4. Run `terraform init`.
5. Run `terraform validate`.
6. Run `terraform plan`.
7. Record the exact resource address shown in the plan.

Expected address:

```text
aws_s3_bucket.example
```

### Lab 2: Inspect state

After applying the configuration in a safe AWS account:

```bash
terraform state list
terraform state show aws_s3_bucket.example
```

Record:

- Resource address
- Provider used
- Physical resource identifier
- Important attributes tracked in state

### Lab 3: Provider dependency recovery

Scenario: a developer runs `terraform init -upgrade` and changes the provider version within the allowed constraint.

Tasks:

1. Inspect `terraform.lock.hcl` before and after the upgrade.
2. Run `terraform plan`.
3. Decide whether the provider upgrade is intentional.
4. If intentional, review and commit the lock-file change.
5. If accidental, restore the approved lock file and run `terraform init`.
6. Re-run `terraform plan` and confirm there is no unexpected infrastructure change.

### Lab 4: Drift recovery

Scenario: change a Terraform-managed property outside Terraform in a controlled test environment.

Tasks:

1. Run `terraform plan` and identify the affected resource address.
2. Determine whether Terraform configuration or the external change is the intended source of truth.
3. Make the smallest safe correction.
4. Run `terraform plan` again.
5. Confirm the final plan matches the intended state.

### Troubleshooting checklist

- What Terraform CLI version is running?
- What provider version is selected?
- Is `.terraform.lock.hcl` present and committed?
- What exact resource address is affected?
- What does `terraform state show <address>` report?
- Is the difference configuration drift, state drift, or a provider/dependency change?
- Does the final plan contain only expected changes?
