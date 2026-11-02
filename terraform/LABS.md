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

### Troubleshooting checklist

- Where did the variable value come from: `-var`, `.tfvars`, or `TF_VAR_*`?
- Does the declared type match the intended value?
- Is validation enforcing the business rule?
- Is a local changing or combining the value?
- What exact resource address consumes the value?
- Is the expected result exposed through a root output?
- Does `terraform console` help isolate the expression?
- Does the final plan contain only expected changes?
