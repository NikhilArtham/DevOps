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

A good module README is a small user guide. Another engineer should be able to understand the module and use it without opening every Terraform file.

Keep the README in the module directory:

~~~text
modules/
└── s3_bucket/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
~~~

### What the README should contain

1. **Purpose** — what the module creates and what it deliberately does not create.
2. **Usage** — a complete, realistic module call that another engineer can copy.
3. **Inputs** — each input's name, type, whether it is required, its default, and what it controls.
4. **Outputs** — each output's name and what a caller can use it for.
5. **Requirements** — Terraform and provider versions, plus provider configuration expectations.
6. **Security and operations** — permissions, encryption, public access, naming, cost, state, and lifecycle considerations.
7. **Validation** — the commands and checks to run before a change is merged.

### Example: modules/s3_bucket/README.md

~~~markdown
# S3 Bucket Module

## Purpose

Creates an S3 bucket with public access blocked, server-side encryption enabled, and optional object versioning.

This module does not configure the AWS provider or create the Terraform state backend. The calling root module owns provider configuration and backend setup.

## Usage

~~~hcl
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
~~~

## Inputs

### Required

- **bucket_name** — `string`; no default. Globally unique S3 bucket name. Use 3–63 lowercase letters, numbers, dots, or hyphens.
- **environment** — `string`; no default. Allowed values: `dev`, `test`, or `prod`.

### Optional

- **tags** — `map(string)`; default: `{}`. Additional tags applied to the bucket. The module sets `Environment` and `ManagedBy` after merging, so those module-owned values take precedence if the caller supplies the same keys.
- **enable_versioning** — `bool`; default: `true`. Enables or suspends S3 bucket versioning according to the module implementation.

## Outputs

- **bucket_id** — bucket name/ID. Useful when another resource needs the bucket identifier.
- **bucket_arn** — bucket ARN. Useful in IAM policies and other AWS resource configurations.

## Requirements

- Terraform version: use the minimum version declared by the root configuration.
- AWS provider: configured by the calling root module; this module does not create provider credentials.
- The caller must have permissions to create and manage the bucket and its security configuration.

## Security and operational notes

- S3 bucket names are globally unique across AWS.
- Public access blocking and server-side encryption are enabled by this module.
- Versioning is enabled by default; confirm the retention and storage-cost implications for the workload.
- Keep Terraform state in a secured remote backend with encryption, access controls, and locking.
- Review the Terraform plan before applying changes, especially changes to encryption, versioning, or bucket identity.

## Validate changes

Run these commands from the root configuration that calls the module:

~~~bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
~~~

Review the plan and confirm that it contains only the intended changes before applying.
~~~


Keep the README synchronized with the actual HCL. If an input, default, output, security behavior, or requirement changes, update this document in the same pull request.

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
