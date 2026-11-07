# DAY 7 — Module Inputs, Outputs, Validation and Documentation

## 1. A module is an API

A reusable Terraform module exposes a public interface:
- **Inputs** are parameters supplied by the caller.
- **Validation** rejects values that violate the module contract.
- **Outputs** return values callers need.
- **Documentation** explains usage, assumptions, and side effects.

Treat variable names, types, defaults, validation rules, and output names as an API. Changing them can break callers. Check known consumers before changing a shared module interface.

## 2. Design inputs deliberately

Example for an S3 bucket module:

```hcl
variable "bucket_name" {
  description = "Globally unique S3 bucket name"
  type        = string
  nullable    = false

  validation {
    condition = (
      length(var.bucket_name) >= 3 &&
      length(var.bucket_name) <= 63 &&
      can(regex("^[a-z0-9][a-z0-9.-]*[a-z0-9]$", var.bucket_name))
    )
    error_message = "Use 3-63 lowercase letters, numbers, dots or hyphens; start and end with a letter or number."
  }
}

variable "environment" {
  description = "Environment that owns this bucket"
  type        = string
  nullable    = false

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}

variable "tags" {
  description = "Additional tags to apply to the bucket"
  type        = map(string)
  default     = {}
}

variable "enable_versioning" {
  description = "Enable S3 object versioning"
  type        = bool
  default     = true
}
```

Design principles:
- Use explicit types: `string`, `number`, `bool`, `list(string)`, `map(string)`, or a typed `object(...)`.
- Give a default only when it is safe and predictable. Keep important environment-specific values required.
- Use `nullable = false` when null is not meaningful.
- Describe purpose, accepted values, units, and constraints.
- Avoid exposing internal implementation details as caller inputs.
- Avoid excessive boolean switches that create many combinations to test.

A type constraint validates the shape of a value; a validation block checks business rules. For example, `type = string` accepts strings, while `contains(...)` restricts the allowed strings.

## 3. Add meaningful validation

### Numeric range

```hcl
variable "retention_days" {
  description = "Number of days to retain objects"
  type        = number
  default     = 30

  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 3650
    error_message = "retention_days must be between 1 and 3650."
  }
}
```

### Collection validation

```hcl
variable "allowed_cidrs" {
  description = "Approved IPv4 CIDR ranges"
  type        = list(string)

  validation {
    condition     = length(var.allowed_cidrs) > 0
    error_message = "Provide at least one approved CIDR range."
  }

  validation {
    condition = alltrue([
      for cidr in var.allowed_cidrs : can(cidrnetmask(cidr))
    ])
    error_message = "Every allowed_cidrs entry must be a valid IPv4 CIDR range."
  }
}
```

A syntactically valid CIDR can still be too broad. Security policy, such as whether `0.0.0.0/0` is acceptable, needs its own explicit rule.

### Validation, preconditions, and postconditions

- **Variable validation** checks caller-provided input.
- **Preconditions** check assumptions before a resource or data-source operation proceeds.
- **Postconditions** check guarantees about the resulting resource or data-source value.
- **Check blocks** can monitor conditions and report warnings; they are not interchangeable with blocking preconditions/postconditions.

Use the narrowest mechanism that expresses the requirement. Do not put checks on inputs if the condition depends on a resource result.

## 4. Outputs are the return contract

```hcl
output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.this.arn
}
```

A parent references a child output as:

```hcl
module.data_bucket.bucket_arn
```

To display it through the root CLI, re-export it:

```hcl
output "data_bucket_arn" {
  description = "ARN of the application data bucket"
  value       = module.data_bucket.bucket_arn
}
```

### Sensitive outputs

```hcl
output "database_password" {
  description = "Generated database password"
  value       = random_password.db.result
  sensitive   = true
}
```

Important: `sensitive = true` redacts normal CLI display; it does not remove the value from Terraform state. Secure the backend and restrict state access. Avoid exposing secrets unless callers genuinely need them.

Output design rules:
- Expose stable values callers consume: IDs, ARNs, endpoints, and names.
- Add descriptions to public outputs.
- Preserve useful types.
- Mark secret-bearing outputs sensitive.
- Avoid exposing every internal attribute.
- Check downstream consumers before removing or renaming outputs.

## 5. Implement the contract in the DAY 6 S3 module

Use the local module from DAY 6. Update `modules/s3_bucket/variables.tf` with `bucket_name`, `environment`, `tags`, and `enable_versioning` as above. Add `environment` to the bucket tags, or merge it with caller tags:

```hcl
locals {
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "Terraform"
  })
}
```

Use `tags = local.common_tags` on `aws_s3_bucket.this`. Later maps passed to `merge` win for duplicate keys, so the module's Environment and ManagedBy values override caller values here. Decide whether that ownership rule is intended and document it.

The child module should export `bucket_id` and `bucket_arn`. The root should pass inputs explicitly and re-export only values its own callers need.

## 6. Document the module for another engineer

A module README should answer:
1. What does the module create, and what does it not create?
2. How do I call it?
3. Which inputs are required and which have defaults?
4. What outputs can I consume?
5. Which provider versions and permissions are needed?
6. What security, cost, naming, and lifecycle considerations apply?
7. How do I validate and test a change?

Example `modules/s3_bucket/README.md`:

```markdown
# S3 Bucket Module

Creates a private S3 bucket with public-access blocking, server-side encryption,
and configurable versioning.

## Usage

\`\`\`hcl
module "data_bucket" {
  source = "./modules/s3_bucket"

  bucket_name       = "company-dev-data-unique-123"
  environment       = "dev"
  enable_versioning = true

  tags = {
    Owner   = "platform"
    Project = "data-platform"
  }
}
\`\`\`

## Inputs

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| bucket_name | string | yes | — | Globally unique bucket name |
| environment | string | yes | — | dev, test, or prod |
| tags | map(string) | no | {} | Additional tags |
| enable_versioning | bool | no | true | Enable object versioning |

## Outputs

| Name | Description |
|---|---|
| bucket_id | Bucket name/ID |
| bucket_arn | Bucket ARN |

## Requirements and notes

- AWS provider configured by the calling root module.
- Bucket names must be globally unique.
- Public access is blocked and server-side encryption is enabled.
- Protect the Terraform state backend and review plans before applying.
```

Keep the README accurate. Update examples, input/output tables, security notes, and tests whenever behavior changes.

## 7. Commands and exact addresses

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform plan -out=tfplan
terraform show tfplan
terraform output
terraform output data_bucket_arn
terraform state list
terraform state show 'module.data_bucket.aws_s3_bucket.this'
```

- `fmt -recursive`: formats HCL in root and nested modules.
- `init`: initializes providers and module sources.
- `validate`: checks configuration structure and types.
- `plan`: evaluates inputs and previews infrastructure changes.
- `plan -out=tfplan`: saves a reviewed plan.
- `show tfplan`: inspects the saved plan.
- `output`: displays root outputs after apply.
- `output <name>`: displays one root output.
- `state list`: lists tracked resource addresses.
- `state show <address>`: inspects one tracked object.

Expected bucket resource address when the module call is `data_bucket`:

```text
module.data_bucket.aws_s3_bucket.this
module.data_bucket.aws_s3_bucket_public_access_block.this
module.data_bucket.aws_s3_bucket_server_side_encryption_configuration.this
module.data_bucket.aws_s3_bucket_versioning.this
```

These are resource addresses. A child output is referenced as `module.data_bucket.bucket_arn`; the root output is displayed with `terraform output data_bucket_arn`.

## 8. Failure scenario — invalid input

A caller supplies `environment = "production"` although the module allows only `dev`, `test`, or `prod`, or passes a malformed bucket name.

Safe recovery:
1. Stop before apply.
2. Identify the module call and exact failing input.
3. Trace the value to tfvars, CLI `-var`, environment variable, workspace variable, or pipeline input.
4. Compare it with the declared type, nullability, and validation.
5. Correct the caller's value; do not weaken a valid module contract to accommodate a typo.
6. Run `terraform fmt -recursive`, `terraform validate`, and `terraform plan`.
7. Confirm there are no unexpected replacements or destroys.

If the contract itself is wrong, update it deliberately, test known callers, document the behavior change, and release a compatible module version.

## 9. Drift scenario — tags changed outside Terraform

An operator changes the bucket's Environment tag directly in AWS while the module input still says `dev`.

Safe recovery:
1. Run `terraform plan` and identify `module.data_bucket.aws_s3_bucket.this`.
2. Inspect it with `terraform state show 'module.data_bucket.aws_s3_bucket.this'`.
3. Check the live tag in AWS and establish who owns it.
4. If Terraform owns the tag, review the plan that restores the declared value.
5. If another system owns it, define that ownership boundary explicitly; do not silently ignore unexplained drift.
6. Re-run plan and confirm only the intended correction remains.
7. Apply through normal review and verify the live value.

Do not delete state to fix a tag difference. State maps Terraform addresses to real infrastructure.

## 10. Common mistakes

- Using weak types such as `any` when the module needs a known structure.
- Giving required inputs misleading defaults.
- Writing validation errors that do not explain how to fix the value.
- Treating `sensitive = true` as encryption or state redaction.
- Forgetting to re-export child outputs at the root.
- Renaming variables or outputs without checking callers.
- Documenting defaults that differ from the HCL.
- Omitting security and lifecycle side effects from documentation.
- Assuming `terraform validate` proves the cloud API operation will succeed.

## 11. Interview-ready mental model

**Inputs define the contract → types define the shape → validation enforces rules → resources implement the capability → outputs return stable values → documentation teaches callers how to use it safely.**

When a module fails, trace the input source, inspect the contract and validation message, then review the plan. For drift, establish ownership before deciding which system should change.
