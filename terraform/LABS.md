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

## DAY 2 — Variables, Locals, Outputs and Type Constraints

### Lab 1: Build a typed module interface

Create a small AWS configuration with variables for `project`, `environment`, and `aws_region`.

Tasks:

1. Give every variable an explicit type.
2. Add validation so `environment` accepts only `dev`, `test`, or `prod`.
3. Run `terraform fmt`.
4. Run `terraform validate`.
5. Run `terraform plan` with `-var="environment=dev"`.
6. Change the value to an invalid environment and observe the validation failure.

### Lab 2: Use locals for naming and tags

Create a local `bucket_name` and a `common_tags` map using the input variables.

Tasks:

1. Use the local bucket name in `aws_s3_bucket.data`.
2. Apply the common tags to the resource.
3. Use `terraform console` to inspect the local expressions.
4. Confirm the exact resource address is `aws_s3_bucket.data`.

### Lab 3: Expose outputs

Create an output for the S3 bucket name.

Run:

```bash
terraform output
terraform output bucket_name
terraform output -raw bucket_name
```

If using a child module, expose the child output again from the root module and confirm the root CLI output shows it.

### Lab 4: Type-constraint failure and recovery

Scenario: the module expects:

```hcl
variable "replicas" {
  type = number
}
```

Provide an invalid value such as `"three"` through a variable source.

Tasks:

1. Run `terraform plan` and capture the type error.
2. Identify where the value was supplied.
3. Fix the input instead of weakening the type constraint.
4. Run `terraform validate` and `terraform plan` again.
5. Confirm the final plan contains only expected changes.

## DAY 3 — Resource Lifecycle, Dependencies and Replace Behavior

### Lab 1: Inspect update vs replacement

Create an AWS resource where one attribute can be changed in place and another test change causes replacement.

Tasks:

1. Run `terraform plan` and identify the exact resource address.
2. Record whether Terraform shows `~`, `-`, `+`, or `-/+`.
3. Identify the exact attribute causing replacement.
4. Run `terraform state show <address>` and compare the stored value with configuration.
5. Save a reviewed plan with `terraform plan -out=tfplan`.
6. Inspect it with `terraform show tfplan`.

### Lab 2: `create_before_destroy`

Use a disposable test resource with:

```hcl
lifecycle {
  create_before_destroy = true
}
```

Tasks:

1. Make a change that requires replacement.
2. Run `terraform plan`.
3. Observe the replacement ordering.
4. Consider whether naming, quotas, or uniqueness constraints could prevent the old and new resources from existing simultaneously.
5. Remove the lifecycle setting after the exercise if it is not part of the intended design.

### Lab 3: Dependency graph

Create two resources where one directly references an attribute of the other.

Tasks:

1. Run `terraform plan`.
2. Identify the implicit dependency.
3. Run `terraform graph`.
4. Add a narrowly scoped `depends_on` only for a dependency Terraform cannot infer.
5. Re-run the plan and confirm the ordering is still correct.

### Lab 4: Accidental replacement recovery

Scenario: a production-like EC2 configuration unexpectedly shows:

```text
-/+ aws_instance.app
```

Tasks:

1. Stop before applying.
2. Identify the exact changed attribute.
3. Determine why the provider requires replacement.
4. Decide whether the change is intentional.
5. If accidental, revert the configuration and run `terraform plan` again.
6. If intentional, evaluate capacity, downtime, data persistence, dependencies, and `create_before_destroy` feasibility.
7. Confirm the final plan before approval.

### Lab 5: Drift and `ignore_changes`

Scenario: an external system changes a Terraform-managed tag.

Tasks:

1. Run `terraform plan` and identify the resource address.
2. Decide whether Terraform or the external system should own the tag.
3. If Terraform owns it, restore the intended configuration.
4. If another system intentionally owns it, evaluate whether `ignore_changes` is appropriate.
5. Re-run `terraform plan` and verify that only expected differences remain.

### Troubleshooting checklist

- What exact resource address is affected?
- Is the plan an update or a replacement?
- Which attribute forces replacement?
- Is the change intentional?
- Can the old and new resources coexist?
- Are there quotas, naming, uniqueness, or dependency constraints?
- Is the dependency implicit or does it genuinely require `depends_on`?
- Is `ignore_changes` hiding an ownership problem?
- Does `terraform state show <address>` match expectations?
- Does the final plan contain only expected changes?


## DAY 4 — `count` vs `for_each` and Stable Resource Addressing

### Lab 1: Create indexed resources with `count`

In a disposable AWS test environment, create three identical resources using a list and `count = length(var.names)`.

Tasks:

1. Use `count.index` to set each resource's Name tag.
2. Run `terraform fmt`, `terraform validate`, and `terraform plan`.
3. Record the addresses: `aws_instance.app[0]`, `aws_instance.app[1]`, and `aws_instance.app[2]`.
4. Apply only in the disposable environment.
5. Run `terraform state list` and `terraform state show 'aws_instance.app[1]'`.

### Lab 2: Reproduce the positional identity problem

Start with the list `["api", "worker", "scheduler"]`. After applying, remove the middle item `worker`.

Tasks:

1. Run `terraform plan` before applying.
2. Record which logical name is now assigned to each index.
3. Inspect the exact addresses and attributes with `terraform state show`.
4. Explain why index 1 now represents scheduler and why index 2 is removed.
5. Do not apply until you can explain every proposed update, replacement, and destroy action.

### Lab 3: Rebuild using `for_each`

Replace the positional list pattern with a set or map keyed by service name.

Tasks:

1. Use `for_each` and `each.key`; for a map, use `each.value` for per-service configuration.
2. Run `terraform plan`.
3. Record addresses such as `aws_instance.service["api"]`.
4. Apply only in the disposable environment.
5. Run `terraform state list` and inspect at least two keyed instances using quoted addresses.

### Lab 4: Migrate addresses safely

Scenario: an existing `count` resource must be converted to `for_each` without replacing existing infrastructure.

Tasks:

1. Back up state using your team's approved state-backup procedure.
2. Run `terraform state list` and inspect each old index with `terraform state show`.
3. Map every old index to the correct logical key.
4. Add a `moved` block for each old-to-new address mapping, or use a carefully controlled `terraform state mv` procedure.
5. Run `terraform plan` and confirm Terraform recognizes address moves rather than unintended destroy/create actions.
6. Do not apply if the mapping is ambiguous or the plan proposes replacing an existing object unexpectedly.

### Troubleshooting checklist

- Is the resource using `count` or `for_each`?
- What exact addresses appear in the plan and state?
- Does a numeric index still represent the same logical object?
- Are `for_each` keys stable and meaningful?
- Which instance is being removed, updated, or replaced?
- Is an address migration required?
- Does each `moved` block map the correct old address to the correct new address?
- Has state been backed up before any direct state migration?
- Does the final plan preserve the intended physical resources?
