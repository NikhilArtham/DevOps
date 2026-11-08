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


## DAY 5 — Dynamic Blocks, Expressions and Conditional Infrastructure

### Lab 1: Test expressions in the console

Use `terraform console` to evaluate:

```hcl
var.environment == "prod" ? "t3.large" : "t3.micro"
[for port in [80, 443, 8080] : port if port != 8080]
merge({ Environment = "dev" }, { Owner = "platform" })
```

Tasks:

1. Explain the result type of each expression.
2. Change the input values and predict the output before running the expression.
3. Use `lookup`, `contains`, and `length` on a small map/list.
4. Use `terraform validate` after putting the expressions into a configuration.

### Lab 2: Conditional resource creation

Create a CloudWatch alarm controlled by `enable_monitoring` and `count = var.enable_monitoring ? 1 : 0`.

Tasks:

1. Plan with the flag set to false and record the planned address/actions.
2. Set it to true and plan again.
3. Record the address `aws_cloudwatch_metric_alarm.high_cpu[0]`.
4. In a disposable environment, apply and inspect it with `terraform state list` and `terraform state show`.
5. Set the flag back to false and inspect the destroy plan. Do not apply unless you intend to remove the alarm.

### Lab 3: Generate nested security-group ingress rules

Use the typed `ingress_rules` variable and dynamic `ingress` block from DAY 5.

Tasks:

1. Supply two approved test rules in a variable file.
2. Run `terraform fmt`, `terraform init`, `terraform validate`, and `terraform plan`.
3. Confirm there is one resource address: `aws_security_group.app`.
4. Inspect the nested ingress configuration and verify each port, protocol, and CIDR.
5. Add a third rule and confirm the plan changes the security group rather than creating another security-group resource.
6. Test an invalid port range and confirm variable validation rejects it.

### Lab 4: Failure and recovery — malformed dynamic input

Scenario: a developer misspells `from_port` as `form_port` or supplies a string instead of a number.

Tasks:

1. Do not apply the configuration.
2. Identify the exact resource address and variable input source.
3. Compare the input object keys and types with the declared object schema.
4. Run `terraform fmt`, `terraform validate`, and `terraform plan`.
5. Correct the input at its source.
6. Review every generated ingress rule for least-privilege CIDRs and required ports.
7. Confirm the final plan contains only intended changes.

### Troubleshooting checklist

- Is this a value expression, resource-level conditional, or nested dynamic block?
- Does the provider schema support the dynamic block label and its arguments?
- Does the input collection have the expected type and object keys?
- Is the dynamic iterator referenced correctly?
- Does a conditional `count = 0` remove an existing resource?
- What exact resource address appears in the plan?
- Are generated ports, protocols, CIDRs, and permissions approved?
- Did validation and plan complete without errors or unexpected changes?


## DAY 6 — Reusable Terraform Module Design

### Lab 1: Build a local S3 child module

Create the root configuration and `modules/s3_bucket/` files from the DAY 6 notes.

Tasks:

1. Put AWS provider configuration in the root module.
2. Define typed child inputs for `bucket_name`, `tags`, and `enable_versioning`.
3. Create the S3 bucket, public-access block, encryption configuration, and versioning resources in the child module.
4. Expose `bucket_id` and `bucket_arn` from the child module.
5. Call it as `module "data_bucket"`.
6. Run `terraform fmt -recursive`, `terraform init`, `terraform validate`, and `terraform plan`.

### Lab 2: Inspect the module plan and state addresses

Tasks:

1. Record all four expected addresses:
   - `module.data_bucket.aws_s3_bucket.this`
   - `module.data_bucket.aws_s3_bucket_public_access_block.this`
   - `module.data_bucket.aws_s3_bucket_server_side_encryption_configuration.this`
   - `module.data_bucket.aws_s3_bucket_versioning.this`
2. Save and inspect a plan using `terraform plan -out=tfplan` and `terraform show tfplan`.
3. Apply only in a disposable AWS account after reviewing the plan.
4. Run `terraform state list` and inspect the bucket with `terraform state show 'module.data_bucket.aws_s3_bucket.this'`.
5. Run `terraform output` and record the bucket ARN.
6. Distinguish the Terraform address from the physical S3 bucket name/ID.

### Lab 3: Reuse the module for a second bucket

Call the same child module a second time with a different module call name and bucket name.

Tasks:

1. Add `module "logs_bucket"` using the same local source.
2. Give it different tags and versioning input.
3. Run `terraform plan`.
4. Record the new addresses under `module.logs_bucket...`.
5. Confirm the original `module.data_bucket...` addresses remain unchanged.
6. Apply only in a disposable account and inspect both module instances in state.

### Lab 4: Module input failure and recovery

Scenario: pass an invalid bucket name or unsupported environment value from the root configuration.

Tasks:

1. Stop before apply.
2. Identify the failing module call and input variable.
3. Inspect the declared type and validation condition in the child module.
4. Correct the value at the caller rather than weakening a valid module contract.
5. Run `terraform fmt -recursive`, `terraform validate`, and `terraform plan`.
6. Confirm the final plan contains only intended resources and actions.

### Lab 5: Drift recovery

Scenario: change the S3 bucket's Environment tag outside Terraform in a controlled test account.

Tasks:

1. Run `terraform plan` and identify the exact module-qualified address.
2. Inspect the bucket using `terraform state show 'module.data_bucket.aws_s3_bucket.this'`.
3. Compare the configuration, Terraform state, and live AWS tag.
4. Decide whether Terraform or the external system owns the tag.
5. If Terraform owns it, review the plan that restores the declared tag.
6. If ownership belongs elsewhere, redesign the interface/ownership deliberately; do not hide unexplained drift with `ignore_changes`.
7. Re-run the plan and verify only the intended correction remains.

### Troubleshooting checklist

- Is this the root module or a child module?
- What exact module call name and child resource address are involved?
- Are required inputs supplied with the expected types?
- Do validation rules reject invalid caller input early?
- Does the child module configure credentials or region that should belong to the root?
- Are outputs sufficient for callers without exposing unnecessary internals?
- Did a module source/version or module call name change?
- Does the plan show address moves, updates, replacements, or destroys?
- Has state ownership been established before responding to drift?
- Does the final plan contain only reviewed changes?


## DAY 7 — Module Inputs, Outputs, Validation and Documentation

### Lab 1: Improve the S3 module input contract

Update the DAY 6 `modules/s3_bucket` module.

Tasks:

1. Add required `environment` input with type `string` and `nullable = false`.
2. Restrict it to `dev`, `test`, or `prod`.
3. Add descriptions and explicit types to all inputs.
4. Keep safe defaults for `tags` and `enable_versioning`; keep `bucket_name` and `environment` required.
5. Add a `retention_days` number variable with a documented valid range, or implement an equivalent numeric validation example.
6. Run `terraform fmt -recursive`, `terraform validate`, and `terraform plan`.

### Lab 2: Expose and consume outputs

Tasks:

1. Ensure the child module exports `bucket_id` and `bucket_arn`.
2. Re-export `bucket_arn` from the root module as `data_bucket_arn`.
3. Run `terraform plan` and inspect output values.
4. Apply only in a disposable account after reviewing the plan.
5. Run `terraform output data_bucket_arn`.
6. Distinguish a module output reference from a resource address.

### Lab 3: Write usable module documentation

Create or update `modules/s3_bucket/README.md`.

Include purpose and resources created, a complete usage example, an inputs table (name/type/required/default/description), outputs and descriptions, provider requirements, security notes, validation commands, and assumptions or lifecycle side effects.

Compare every documented default and output with the actual HCL.

### Lab 4: Reproduce invalid-input failure

Scenario: call the module with `environment = "production"` or a malformed bucket name.

Tasks:

1. Run `terraform plan` and capture the validation error.
2. Trace the input to its actual source.
3. Explain why type checking and validation are separate.
4. Correct the caller input without weakening the module contract.
5. Re-run format, validate, and plan.
6. Confirm no unexpected creates, replacements, or destroys.

### Lab 5: Reconcile tag drift

Scenario: change the bucket's Environment tag directly in AWS in a disposable environment.

Tasks:

1. Run plan and identify `module.data_bucket.aws_s3_bucket.this`.
2. Inspect the resource using `terraform state show`.
3. Compare declared input, state, and live AWS value.
4. Decide whether Terraform or the external system owns the tag.
5. Restore the intended value or redesign ownership explicitly.
6. Re-run plan and verify only the intended change remains.

### Troubleshooting checklist

- Which caller supplied the value?
- Does it match the declared type and nullability?
- Did validation fail, or did the provider reject an otherwise valid value?
- Is the validation message actionable?
- Is the output declared in the child module and re-exported if needed?
- Is a sensitive output still stored in state?
- Does the README match current inputs, defaults, outputs, and behavior?
- What exact module-qualified resource address is affected?
- Who owns the drifted attribute?
- Does the final plan contain only intended changes?


## DAY 8 — Provider Aliases and Multi-Account Patterns

### Lab 1: Configure two AWS provider aliases

In a sandbox organization with two approved AWS accounts, configure `aws.dev` and `aws.prod` in the root module. Use separate cross-account execution roles and the same test region.

Tasks:

1. Configure the AWS provider source/version once in `required_providers`.
2. Configure both aliases with their approved role ARNs.
3. Do not place access keys or secrets in HCL or commit them to Git.
4. Add `aws_caller_identity` data sources using `provider = aws.dev` and `provider = aws.prod`.
5. Run `terraform init`, `terraform validate`, and `terraform plan`.
6. Verify that the returned account IDs match the approved dev and prod account inventory before proceeding.

Expected data-source addresses:

```text
data.aws_caller_identity.dev
data.aws_caller_identity.prod
```

### Lab 2: Route resources to the correct account

Create one disposable S3 bucket per account using explicit provider selection.

Tasks:

1. Set `provider = aws.dev` on the dev bucket.
2. Set `provider = aws.prod` on the prod bucket.
3. Run `terraform plan -out=tfplan` and inspect it with `terraform show tfplan`.
4. Record the addresses `aws_s3_bucket.dev` and `aws_s3_bucket.prod`.
5. Confirm the account IDs and region for both provider configurations.
6. Apply only after review and only in approved sandbox accounts.
7. Run `terraform state list` and inspect both resources with `terraform state show`.

### Lab 3: Pass aliases into a reusable child module

Use one child module that declares `configuration_aliases = [aws.target]` and creates one S3 bucket through `aws.target`.

Tasks:

1. Call the module as `module.dev_bucket` and map `aws.target = aws.dev`.
2. Call the same source as `module.prod_bucket` and map `aws.target = aws.prod`.
3. Run `terraform providers`, `terraform validate`, and `terraform plan`.
4. Record `module.dev_bucket.aws_s3_bucket.this` and `module.prod_bucket.aws_s3_bucket.this`.
5. Confirm the module source is reusable and the root module controls the account selection.
6. Apply only in disposable accounts after checking identity and plan.

### Lab 4: Reproduce wrong-provider selection and recover

Scenario: add a resource intended for production but omit its explicit provider mapping, or deliberately map the module to the dev provider.

Tasks:

1. Do not apply the incorrect plan.
2. Inspect the provider argument or module `providers` map.
3. Verify each alias with `aws_caller_identity`.
4. Correct the provider mapping.
5. Run `terraform plan` again and verify the destination account and exact resource addresses.
6. Explain why an alias name alone is not proof of the target account.

### Lab 5: Troubleshoot AssumeRole access denied

Scenario: Terraform cannot assume the target account role.

Tasks:

1. Identify the source AWS identity.
2. Check the source identity policy for `sts:AssumeRole` on the exact role ARN.
3. Check the target role trust policy.
4. Check required external ID, MFA, session tag, or organization conditions.
5. Check the assumed role's permissions, SCPs, and permission boundaries.
6. Fix only the missing policy condition or permission; do not grant blanket administrator access.
7. Re-run identity verification and plan.

### Lab 6: Provider mapping drift / account mismatch

Scenario: a provider mapping or backend/workspace selection changes and Terraform proposes unexpected resource actions.

Tasks:

1. Stop before apply.
2. Verify account ID and region for each alias.
3. Inspect the exact state addresses and live objects.
4. Compare configuration, state, and remote infrastructure.
5. Determine whether the provider mapping, state backend, workspace, or physical resource ownership changed.
6. Restore the approved mapping or design a separately reviewed account migration.
7. Do not delete state or blindly run `state rm`.
8. Re-run plan and confirm only approved actions remain.

### Troubleshooting checklist

- Which provider alias does each resource or module use?
- What account ID and region does each alias actually resolve to?
- Is the root module passing the intended provider configuration?
- Has the child module declared the aliased provider in `configuration_aliases`?
- Does the source identity have `sts:AssumeRole` permission?
- Does the target role trust the source identity?
- Does the assumed role have the required service permissions?
- Could an SCP, permissions boundary, explicit deny, or session condition block the call?
- Did the backend or workspace change?
- Are resource addresses and physical resource ownership understood?
- Does the final plan target only the intended account and region?
