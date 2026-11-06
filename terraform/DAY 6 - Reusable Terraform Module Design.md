# DAY 6 — Reusable Terraform Module Design

## 1. What a reusable module solves

A Terraform module is a collection of Terraform configuration files in one directory. Every Terraform configuration has a root module; a directory called by another module becomes a child module.

A reusable module packages a repeatable infrastructure pattern behind a clear interface:

- **Inputs** let callers provide environment-specific values.
- **Resources** implement the infrastructure.
- **Outputs** expose useful results without requiring callers to know every internal detail.
- **Validation and sensible defaults** prevent invalid or unsafe configurations.
- **Documentation and examples** explain how to use the module.

A module is not automatically reusable just because its code is in a separate folder. Reusability comes from a stable interface, sensible boundaries, predictable behavior, and clear ownership.

## 2. A practical module directory structure

For a small root configuration that calls an S3 bucket module:

```text
terraform/
├── main.tf
├── providers.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    └── s3_bucket/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

The root module owns provider configuration and calls the child module. The child module declares the resources and its input/output contract.

For a larger production module, also consider:
- `versions.tf` for Terraform and provider requirements.
- `README.md` for usage, inputs, outputs, examples, and limitations.
- `examples/` for realistic caller configurations.
- Tests and CI checks for formatting, validation, and plan review.

Do not create a separate module for every tiny resource by default. A module should represent a useful capability or ownership boundary, not merely add another layer of indirection.

## 3. Root module versus child module

**Root module**
- Configures the AWS provider and credentials.
- Selects the environment and supplies values.
- Calls child modules.
- Connects module outputs to other root resources or outputs.
- Owns the deployment workflow and state for that configuration.

**Child module**
- Declares variables, resources, locals, and outputs for its capability.
- Should normally inherit provider configuration from its caller.
- Should not contain environment-specific credentials.
- Should avoid hardcoding account IDs, regions, names, or environment assumptions.

A child module is not a separate Terraform state by default. Its resources are tracked in the caller's state with module-qualified addresses.

## 4. Build a reusable S3 bucket module

This example creates a private S3 bucket with public-access blocking, server-side encryption, and versioning. It is intentionally small enough to understand but demonstrates a useful module contract.

### Child module: `modules/s3_bucket/variables.tf`

```hcl
variable "bucket_name" {
  description = "Globally unique name for the S3 bucket"
  type        = string

  validation {
    condition = (
      length(var.bucket_name) >= 3 &&
      length(var.bucket_name) <= 63 &&
      can(regex("^[a-z0-9][a-z0-9.-]*[a-z0-9]$", var.bucket_name))
    )
    error_message = "bucket_name must be 3-63 characters, use lowercase letters, numbers, dots or hyphens, and start/end with a letter or number."
  }
}

variable "tags" {
  description = "Tags applied to every resource that supports tags"
  type        = map(string)
  default     = {}
}

variable "enable_versioning" {
  description = "Whether object versioning should be enabled"
  type        = bool
  default     = true
}
```

The interface is explicit: callers must provide a bucket name; tags and versioning have defaults. Validation catches a common class of bad input before an API request is made. S3 bucket naming has additional AWS-specific restrictions, so this simplified validation is not a complete substitute for AWS validation.

### Child module: `modules/s3_bucket/main.tf`

```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = var.tags
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}
```

Notice the references to `aws_s3_bucket.this.id`. They create implicit dependencies: Terraform knows the bucket must exist before it can configure the related bucket settings. Avoid adding `depends_on` when a direct reference already expresses the dependency.

### Child module: `modules/s3_bucket/outputs.tf`

```hcl
output "bucket_id" {
  description = "Name/ID of the created S3 bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.this.arn
}
```

Outputs form the child module's return contract. Expose values callers genuinely need; do not expose every internal attribute without a reason.

## 5. Call the module from the root configuration

### Root: `main.tf`

```hcl
module "data_bucket" {
  source = "./modules/s3_bucket"

  bucket_name        = var.bucket_name
  enable_versioning  = true

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "data-platform"
  }
}
```

### Root: `variables.tf`

```hcl
variable "bucket_name" {
  description = "Unique bucket name for this environment"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be dev, test, or prod."
  }
}
```

### Root: `outputs.tf`

```hcl
output "data_bucket_name" {
  value = module.data_bucket.bucket_id
}

output "data_bucket_arn" {
  value = module.data_bucket.bucket_arn
}
```

### Root: `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

The provider version here is illustrative; use the version approved by your project and commit the dependency lock file. In a reusable child module, declare provider requirements as needed, but normally configure the provider in the root and pass/inherit it rather than hardcoding credentials or a region in the child.

### Root: `terraform.tfvars`

```hcl
bucket_name = "nikhil-dev-data-unique-12345"
environment = "dev"
```

The bucket name must be globally unique across AWS. Use a name you control in a disposable account for the lab.

## 6. Exact resource addresses and module state

A module call named `data_bucket` prefixes the child resources with `module.data_bucket`.

Expected addresses:

```text
module.data_bucket.aws_s3_bucket.this
module.data_bucket.aws_s3_bucket_public_access_block.this
module.data_bucket.aws_s3_bucket_server_side_encryption_configuration.this
module.data_bucket.aws_s3_bucket_versioning.this
```

These are Terraform resource addresses, not physical AWS IDs. The physical bucket name/ID is exposed by the child output and can also be inspected in state.

Useful commands:

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform plan -out=tfplan
terraform show tfplan
terraform state list
terraform state show 'module.data_bucket.aws_s3_bucket.this'
terraform output
terraform output data_bucket_arn
```

- `fmt -recursive`: formats Terraform files in the current directory and subdirectories.
- `init`: initializes the root configuration, providers, and local child-module references.
- `validate`: checks configuration syntax and internal consistency.
- `plan`: previews changes; it does not apply them.
- `plan -out=tfplan`: saves the reviewed plan to a file.
- `show tfplan`: displays the saved plan.
- `state list`: lists exact addresses tracked in the current state.
- `state show <address>`: inspects one resource instance.
- `output`: displays root outputs.

Run `terraform state show` only after the resources have been applied to a safe test account. Before apply, use the plan to inspect intended values.

## 7. Module design decisions that matter in production

### Keep inputs intentional

Use typed variables and validation for important constraints. Avoid dozens of switches that make the module difficult to understand. Prefer a small set of inputs that represent real decisions a caller should control.

### Keep provider configuration at the correct layer

The root module usually configures the provider, credentials, region, and aliases. A child module should not embed credentials. If a child needs a specific provider alias, declare and pass that provider explicitly from the caller.

### Use stable module sources

- Local source: `./modules/s3_bucket` — useful when module code lives in the same repository.
- Git source: a repository URL and a pinned ref — useful for shared modules.
- Registry source: a published module with a selected version — useful for centrally maintained modules.

For Git or registry modules, pin a known release/ref in production rather than consuming a moving branch. Re-run `terraform init` after changing module source or version.

### Design outputs as a contract

Outputs should expose stable, useful information such as IDs, ARNs, DNS names, or endpoints. Root modules can consume them through `module.<call_name>.<output_name>`.

### Understand state ownership

A child module called from a root module normally shares the root's state. Renaming a module call changes addresses from, for example, `module.data_bucket...` to `module.archive_bucket...`. Terraform may propose destroy/create unless the address migration is declared with an appropriate `moved` block or performed through a controlled state migration. Never rename a module call casually in a production deployment.

## 8. Realistic failure scenario — invalid module input

A platform team creates a shared S3 module. An application team calls it with a bucket name containing uppercase characters or an invalid environment value.

Symptoms:
- `terraform validate` or `terraform plan` reports a variable validation failure.
- No infrastructure change should be applied from the invalid configuration.
- The error should identify the input contract that was violated.

Safe recovery:
1. Stop before apply and identify the module call and failing input.
2. Check the value in the root tfvars file, workspace variable set, or pipeline input.
3. Compare it with the child module's declared type and validation rules.
4. Correct the value at the caller; do not weaken module validation just to make the run pass.
5. Run `terraform fmt -recursive`, `terraform validate`, and `terraform plan` again.
6. Confirm the plan contains only the intended module resources and no unrelated replacement or destroy actions.

If the module's validation rule itself is incorrect, update the module contract deliberately, test callers, and review the resulting plan before release.

## 9. Realistic drift scenario — out-of-band tag change

An operator changes the S3 bucket's `Environment` tag directly in AWS while Terraform still declares `Environment = "dev"`.

Safe recovery:
1. Run `terraform plan` and identify the exact module-qualified address and changed tag.
2. Inspect the resource using `terraform state show 'module.data_bucket.aws_s3_bucket.this'`.
3. Check the live bucket tags in AWS and determine whether the manual change was intentional.
4. If Terraform is the source of truth, retain the intended tag in module inputs and review the plan that restores it.
5. If another system owns that tag, redesign ownership explicitly; do not casually add `ignore_changes` to hide unexplained drift.
6. Re-run the plan and verify that only the intended tag correction remains.
7. Apply only through the normal review/change process, then verify the live tag.

Do not delete state or import an already-managed resource as a generic drift fix. The module resource is already tracked; reconcile the declared configuration with the intended live value.

## 10. Common module design mistakes

- Creating modules that are only thin wrappers around one resource without adding a useful contract or abstraction.
- Hardcoding account-specific IDs, regions, names, or credentials inside child modules.
- Putting provider credentials or root-level provider configuration in a reusable child module.
- Exposing too many options and making every caller configure every detail.
- Returning no useful outputs, forcing callers to depend on internal implementation.
- Changing module call names or resource labels without planning state-address migration.
- Using broad `ignore_changes` rules to suppress drift instead of resolving ownership.
- Treating `terraform validate` as proof that AWS will accept the request.
- Applying a module change without reviewing all resources under its module-qualified address.

## 11. AWS and Kubernetes relevance

AWS:
- Shared S3, VPC, security-group, IAM-role, CloudWatch, and RDS patterns.
- Standardized tags, encryption, logging, network controls, and naming.
- Environment-specific root configurations calling the same approved module.

Kubernetes:
- Reusable modules can package namespaces, service accounts, deployments, services, and common labels.
- Keep cluster/provider configuration in the root layer and expose stable outputs such as namespace names or service details.
- Avoid building a module so generic that callers cannot understand what Kubernetes objects it will create.

## 12. Interview-ready mental model

**Root module configures providers and supplies inputs → child module validates its contract → resources implement the capability → outputs return useful values → root state tracks module-qualified resource addresses.**

When troubleshooting a module, inspect the caller's inputs, the child module contract, the exact module-qualified address, and the plan before changing state or applying infrastructure.
