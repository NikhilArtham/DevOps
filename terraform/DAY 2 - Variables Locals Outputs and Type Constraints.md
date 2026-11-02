# DAY 2 — Variables, Locals, Outputs and Type Constraints

## 1. Why these concepts matter

Terraform configuration becomes reusable when values are not hard-coded everywhere. Variables provide inputs, locals calculate or standardize values inside a module, outputs expose useful results, and type constraints prevent invalid values from reaching resources.

A practical mental model is:

`Input variable → validation/type check → locals → resources → outputs`

These are Terraform language constructs. They do not create AWS or Kubernetes objects by themselves. They control the values used to configure resources and the information exposed after Terraform evaluates the configuration.

---

## 2. Input variables

A variable is an input to a Terraform module.

Example:

```hcl
variable "aws_region" {
  description = "AWS region where resources are created"
  type        = string
  default     = "ap-south-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}
```

Use it with:

```hcl
provider "aws" {
  region = var.aws_region
}
```

and:

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "nikhil-${var.environment}-example"
}
```

Common ways to provide variable values:

- `terraform.tfvars`
- `*.auto.tfvars`
- `-var='environment=dev'`
- `-var-file=dev.tfvars`
- environment variables such as `TF_VAR_environment`

Do not commit secrets into `.tfvars` files. Use an appropriate secret-management mechanism for sensitive values.

### Variable validation

Type constraints catch broad type errors. Validation can enforce business rules.

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "Environment must be dev, test, or prod."
  }
}
```

`type` answers: "What kind of value is this?"

`validation` answers: "Is this particular value acceptable?"

---

## 3. Type constraints

Type constraints make module interfaces predictable.

### Primitive types

```hcl
variable "environment" {
  type = string
}

variable "replicas" {
  type = number
}

variable "enable_logging" {
  type = bool
}
```

### Collection types

```hcl
variable "availability_zones" {
  type = list(string)
}

variable "tags" {
  type = map(string)
}

variable "allowed_ports" {
  type = set(number)
}
```

### Object types

Object constraints are especially useful when a module accepts structured configuration.

```hcl
variable "application" {
  type = object({
    name     = string
    port     = number
    replicas = number
  })
}
```

Example value:

```hcl
application = {
  name     = "orders-api"
  port     = 8080
  replicas = 3
}
```

For optional object attributes, Terraform supports `optional(...)` in supported Terraform versions:

```hcl
variable "application" {
  type = object({
    name     = string
    port     = number
    replicas = optional(number, 2)
  })
}
```

### Tuple vs list

A `list(string)` represents a collection where every element has the same type.

A tuple can contain different types and has a fixed positional shape, for example:

```hcl
tuple([string, number])
```

Use the simplest type that accurately describes the module contract.

---

## 4. Locals

Locals create named expressions that are reused inside a module.

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Use them with:

```hcl
tags = local.common_tags
```

Locals are useful for:

- common tags
- naming conventions
- derived values
- repeated expressions
- conditional values
- reducing duplication

A local is not an input. It is calculated from configuration available to the module.

Avoid creating locals that merely rename a value once. The goal is clearer and more maintainable configuration, not unnecessary abstraction.

---

## 5. Outputs

Outputs expose values from a module.

```hcl
output "bucket_name" {
  description = "Name of the application S3 bucket"
  value       = aws_s3_bucket.example.bucket
}
```

After apply, inspect it with:

```bash
terraform output
terraform output bucket_name
terraform output -raw bucket_name
```

Outputs are useful when:

- a root module needs to expose infrastructure information
- a parent module needs a child module's value
- CI/CD needs a value after deployment
- another Terraform configuration consumes a remote module output

Example module usage:

```hcl
module "app" {
  source      = "./modules/app"
  environment = "prod"
}

output "app_bucket" {
  value = module.app.bucket_name
}
```

Do not treat outputs as a general secret store. If an output contains sensitive information, mark it appropriately and understand that sensitive values can still exist in Terraform state.

```hcl
output "database_password" {
  value     = var.database_password
  sensitive = true
}
```

---

## 6. Variables vs locals vs outputs

| Construct | Purpose | Direction |
|---|---|---|
| Variable | Accept input | Into module |
| Local | Calculate/reuse internal values | Inside module |
| Output | Expose a result | Out of module |
| Type constraint | Define allowed value shape | Validates input/interface |

Example:

```text
var.environment
      ↓
local.name_prefix
      ↓
resource.aws_s3_bucket.example
      ↓
output.bucket_name
```

---

## 7. Commands directly used with these concepts

### `terraform fmt`

Formats Terraform configuration.

```bash
terraform fmt
```

### `terraform validate`

Checks whether the configuration is syntactically valid and internally consistent.

```bash
terraform validate
```

It is useful for catching incorrect variable types, references, and configuration structure before planning.

### `terraform plan`

Evaluates variables, locals, resources, and outputs as part of the execution plan.

```bash
terraform plan
```

Useful variable-related example:

```bash
terraform plan -var="environment=prod"
```

### `terraform output`

Reads root-module outputs from state.

```bash
terraform output
terraform output bucket_name
terraform output -raw bucket_name
```

### `terraform console`

Useful for testing expressions involving variables, locals, functions, and types.

```bash
terraform console
```

Examples:

```text
> upper("prod")
"PROD"

> type({name = "api", replicas = 3})
object({name: string, replicas: number})
```

---

## 8. AWS objects directly used by this pattern

Variables, locals, outputs, and type constraints are not AWS resources. They configure or expose values used by AWS resources.

For this topic, a small example can use:

- `aws_s3_bucket`
- `aws_s3_bucket_versioning`
- AWS provider region
- resource tags

Example:

```hcl
variable "environment" {
  type = string
}

variable "project" {
  type = string
}

locals {
  bucket_name = "${var.project}-${var.environment}-data"

  tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "data" {
  bucket = local.bucket_name
  tags   = local.tags
}

output "bucket_name" {
  value = aws_s3_bucket.data.bucket
}
```

The important resource address is:

```text
aws_s3_bucket.data
```

The output is derived from that resource and does not create another AWS object.

---

## 9. Kubernetes relevance

The same Terraform language concepts are commonly used with the Kubernetes provider.

For example, a variable can control a deployment replica count:

```hcl
variable "replicas" {
  type = number
}

resource "kubernetes_deployment" "app" {
  # deployment configuration uses var.replicas
}
```

The Terraform constructs remain the same; the provider determines which external platform object is managed.

---

## 10. Realistic failure scenario — wrong variable type reaches a production plan

### Situation

A module expects:

```hcl
variable "replicas" {
  type = number
}
```

A deployment pipeline passes:

```text
TF_VAR_replicas="three"
```

Terraform cannot use the string `three` where a number is required.

### What you should do

Run:

```bash
terraform validate
terraform plan
```

Inspect the variable declaration and the value source. Check whether the value came from a `.tfvars` file, `-var`, or `TF_VAR_` environment variable.

Correct the input to a numeric value:

```text
TF_VAR_replicas=3
```

Then run the plan again and verify that the resulting resource changes are exactly what you expect.

### Production lesson

Do not solve a type error by weakening the type constraint just to make the pipeline pass. Strong module contracts prevent bad configuration from reaching infrastructure.

---

## 11. Another common failure — output exists but is not visible

A developer creates an output in a child module:

```hcl
output "bucket_name" {
  value = aws_s3_bucket.data.bucket
}
```

But the root module does not expose it. Running `terraform output` from the root will not show the child output automatically.

The root module must explicitly expose it:

```hcl
output "app_bucket" {
  value = module.app.bucket_name
}
```

This is an important interview distinction: child-module outputs are available to the parent through `module.<name>.<output>`, but root CLI output requires a root output.

---

## 12. Common mistakes

- Hard-coding environment-specific values instead of using variables.
- Using `any` when a precise type constraint is practical.
- Confusing a local with an input variable.
- Assuming outputs create resources.
- Forgetting that root outputs must be declared in the root module to appear in `terraform output`.
- Putting secrets directly in Git-tracked `.tfvars` files.
- Assuming `sensitive = true` removes a value from Terraform state.
- Weakening a type constraint instead of fixing a bad input.
- Creating too many locals and making simple expressions harder to understand.

---

## 13. Interview-ready mental model

**Variables are the module's API inputs.**

**Type constraints define the shape of those inputs.**

**Locals are internal calculated/reusable values.**

**Resources consume those values to manage infrastructure.**

**Outputs expose selected results to callers, CI/CD, or users.**

In a production troubleshooting scenario, start by finding where the value entered Terraform, confirm its type and validation, inspect the local expression if one is involved, identify the exact resource address affected, and review the output only if the problem is about an exposed result.
