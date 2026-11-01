# DAY 1 — Terraform Professional Workflow: Repo Structure, Providers and Versions

## 1. Why Terraform projects need a predictable structure

A professional Terraform repository separates configuration by responsibility instead of keeping everything in one large `.tf` file.

A small project can start with:

```text
terraform-project/
├── versions.tf
├── providers.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── .gitignore
└── README.md
```

A larger production repository may use reusable modules and environment-specific root modules:

```text
terraform/
├── modules/
│   ├── network/
│   ├── compute/
│   └── database/
├── environments/
│   ├── dev/
│   ├── test/
│   └── prod/
└── README.md
```

The important principle is that Terraform files in the same root module are loaded together. Splitting resources across `main.tf`, `network.tf`, or `database.tf` is mainly for human organization; Terraform does not execute the files in filename order.

## 2. `terraform` block and version constraints

Terraform itself and providers are separate dependencies. Pin both intentionally.

Example:

```hcl
terraform {
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

### What these constraints mean

- `required_version` controls which Terraform CLI versions can run the configuration.
- `source = "hashicorp/aws"` identifies the provider namespace and provider name.
- `version = "~> 6.0"` allows compatible 6.x releases but prevents automatic movement to 7.x.

Avoid blindly using `latest`-style behavior. A production repository should make dependency changes deliberate and reviewable.

## 3. Provider configuration

A provider supplies the implementation Terraform uses to communicate with an external platform such as AWS.

Example:

```hcl
provider "aws" {
  region = var.aws_region
}
```

Keep provider configuration separate from resource definitions when that improves readability.

Do not hard-code long-lived cloud credentials into `.tf` files. Prefer AWS IAM roles, environment credentials, GitHub Actions OIDC, or another supported credential mechanism.

## 4. Provider version versus lock file

Terraform uses the dependency constraints in the configuration to determine acceptable versions. After dependency resolution, Terraform records the selected provider versions and checksums in `.terraform.lock.hcl`.

Example workflow:

```bash
terraform init
terraform providers
terraform version
```

Commit `.terraform.lock.hcl` to version control for a team repository. Do **not** commit the `.terraform/` directory or state files unless there is a very specific reason and an appropriate security design.

Typical `.gitignore` entries:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.tfvars.json
```

Review whether a particular `.tfvars` file actually contains secrets before ignoring or committing it. Sensitive values should normally come from a secure secret-management mechanism.

## 5. Small professional configuration

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

`providers.tf`:

```hcl
provider "aws" {
  region = var.aws_region
}
```

`variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region used by this configuration"
  type        = string
  default     = "ap-south-1"
}
```

`main.tf`:

```hcl
resource "aws_s3_bucket" "example" {
  bucket_prefix = "nikhil-terraform-demo-"
}
```

The resource address is:

```text
aws_s3_bucket.example
```

That exact address matters when reading plans, state, imports, replacements, and targeted troubleshooting.

## 6. Inspecting the plan and state

Initialize the working directory:

```bash
terraform init
```

Validate syntax and configuration:

```bash
terraform validate
```

Create a plan:

```bash
terraform plan
```

Inspect the resource address in state:

```bash
terraform state list
terraform state show aws_s3_bucket.example
```

Useful mental model:

- Configuration defines the desired infrastructure.
- Provider implements communication with the platform.
- Plan compares configuration, state, and real infrastructure.
- State maps Terraform resource addresses to real infrastructure objects.

## 7. Failure scenario: provider version drift

Suppose one developer runs `terraform init -upgrade` and receives a newer provider version within the allowed constraint. Another developer then gets a different provider version because the lock file was not committed.

This can produce inconsistent plans or provider behavior between machines.

Safe recovery:

1. Check the configured constraint in `versions.tf`.
2. Inspect `.terraform.lock.hcl`.
3. Run `terraform providers` and `terraform version`.
4. If the newer version is intentionally approved, regenerate/update the lock file and review the resulting plan.
5. If the upgrade was accidental, restore the expected lock file and run `terraform init` again.
6. Run `terraform plan` and verify that unexpected infrastructure changes are absent.

Do not solve dependency drift by deleting state. Terraform state represents infrastructure ownership and identity; provider dependency problems should normally be fixed at the dependency/lock-file level.

## 8. Failure scenario: resource drift

After creating the S3 bucket, someone changes a property outside Terraform or the infrastructure is modified by another system.

Run:

```bash
terraform plan
```

Terraform may report that the real object differs from the configuration/state expectation.

Safe recovery depends on the desired source of truth:

- If Terraform should own the setting, update the configuration if necessary and apply the intended change.
- If the external change was valid and should become the new desired configuration, update Terraform configuration to match it and then plan again.
- Do not manually edit the state file as a first response.

## 9. Common mistakes

- Putting provider credentials directly in Terraform files.
- Using an unbounded provider version.
- Forgetting to commit `.terraform.lock.hcl`.
- Committing `.terraform/` or `*.tfstate` to Git.
- Assuming `.tf` files execute alphabetically.
- Treating Terraform state as disposable cache.
- Using `terraform apply -auto-approve` in production without an appropriate review/control process.
- Ignoring the exact resource address when troubleshooting a plan.

## 10. Interview-ready mental model

**Terraform CLI version → configuration constraints → provider source/version → lock file → initialization → plan → resource address → state → real infrastructure.**

If asked why provider versions are pinned, the practical answer is: **to make infrastructure changes reproducible and prevent an unreviewed provider upgrade from changing behavior across environments or CI runs.**
