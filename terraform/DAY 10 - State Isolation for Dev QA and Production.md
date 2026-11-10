# DAY 10 — State Isolation for Dev, QA and Production

## 1. The problem state isolation solves

Terraform state maps Terraform resource addresses to real infrastructure objects. If dev, QA, and production share one state, a change intended for one environment can affect another environment's resources, approvals, or lifecycle.

The goal is to reuse the same Terraform configuration while giving each environment an independent state boundary.

A useful mental model:

- **Configuration** describes what each environment should look like.
- **Backend key** identifies which remote state object Terraform reads and writes.
- **Credentials/provider configuration** determine which AWS account and region Terraform can manage.
- **State** records the objects managed by that particular root configuration.

Separate state keys isolate state operations, but they do not by themselves prevent a role from accessing another environment's AWS resources or state. Use separate AWS accounts and restricted roles for stronger production isolation.

## 2. Isolation options

| Pattern | How it isolates | Important limitation |
|---|---|---|
| CLI workspaces | Separate state instance per workspace | Same backend and usually same credentials; not a security boundary |
| Separate backend keys | Separate S3 state objects | IAM must still restrict access to each key |
| Separate accounts + backend keys/roles | State, credentials, and account boundary | More setup, but strongest production separation |

For dev/QA/prod with different approval and access requirements, prefer separate root execution contexts, backend keys, and AWS roles. Production should normally live in a separate AWS account.

## 3. Example repository layout

One practical layout is:

```text
terraform-environments/
├── modules/
│   └── app-bucket/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── backend.hcl
│   │   └── terraform.tfvars
│   ├── qa/
│   │   ├── backend.hcl
│   │   └── terraform.tfvars
│   └── prod/
│       ├── backend.hcl
│       └── terraform.tfvars
└── root/
    ├── versions.tf
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

The root configuration is reused, while each environment supplies its own backend configuration and variable values. Keep credentials out of these files; use profiles, assumed roles, or CI workload identity.

## 4. Small working configuration

This exercise creates one S3 bucket per environment. Use a unique bucket name for each environment and a disposable AWS account. The backend bucket must already exist.

### root/versions.tf

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {}
}

provider "aws" {
  region = var.aws_region
}

```

### root/variables.tf

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "qa", "prod"], var.environment)
    error_message = "environment must be dev, qa, or prod."
  }
}

variable "bucket_name" {
  description = "Globally unique S3 bucket name for this environment"
  type        = string
}

variable "aws_region" {
  description = "AWS region for the managed bucket"
  type        = string
  default     = "ap-south-1"
}
```

### root/main.tf

```hcl
resource "aws_s3_bucket" "app" {
  bucket = var.bucket_name

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

### root/outputs.tf

```hcl
output "bucket_name" {
  value = aws_s3_bucket.app.bucket
}
```

The root module has the same resource address in every environment:

```text
aws_s3_bucket.app
```

That is expected. Resource addresses are relative to a particular state. The dev state and prod state can each independently contain `aws_s3_bucket.app`, mapping it to different physical bucket names.

## 5. Environment-specific backend configuration

Use the same backend bucket but a different key for each environment.

### environments/dev/backend.hcl

```hcl
bucket       = "company-terraform-state"
key          = "application/dev/ap-south-1/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

### environments/qa/backend.hcl

```hcl
bucket       = "company-terraform-state"
key          = "application/qa/ap-south-1/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

### environments/prod/backend.hcl

```hcl
bucket       = "company-terraform-state"
key          = "application/prod/ap-south-1/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

The state and lock objects are separate:

```text
application/dev/ap-south-1/terraform.tfstate
application/dev/ap-south-1/terraform.tfstate.tflock

application/qa/ap-south-1/terraform.tfstate
application/qa/ap-south-1/terraform.tfstate.tflock

application/prod/ap-south-1/terraform.tfstate
application/prod/ap-south-1/terraform.tfstate.tflock
```

Backend blocks cannot use normal Terraform input variables or locals. Supplying a different static backend file at initialization is the standard pattern.

### Environment variable files

`environments/dev/terraform.tfvars`:

```hcl
environment = "dev"
bucket_name = "nikhil-app-dev-unique-12345"
aws_region  = "ap-south-1"
```

`environments/qa/terraform.tfvars`:

```hcl
environment = "qa"
bucket_name = "nikhil-app-qa-unique-12345"
aws_region  = "ap-south-1"
```

`environments/prod/terraform.tfvars`:

```hcl
environment = "prod"
bucket_name = "nikhil-app-prod-unique-12345"
aws_region  = "ap-south-1"
```

Replace the example bucket names with globally unique names before planning. Never reuse one bucket name across environments.

## 6. Initialize and plan each environment

Run commands from the root configuration directory, passing the desired environment files explicitly. Use separate working directories or isolated CI jobs for each environment so that cached backend metadata in `.terraform` cannot be accidentally reused.

Example dev initialization and plan:

```bash
terraform -chdir=root init -backend-config=../environments/dev/backend.hcl
terraform -chdir=root plan -var-file=../environments/dev/terraform.tfvars -out=dev.tfplan
terraform -chdir=root show dev.tfplan
```

For QA and production, use their respective backend and variable files in separate clean working directories or clean CI workspaces. Do not run multiple environment initializations concurrently in the same working directory.

A production pipeline should also assume the production role only after the required approval. Verify the account identity before planning or applying:

```bash
aws sts get-caller-identity
```

Compare the returned account ID with the approved environment account. Do not trust the directory name alone.

## 7. Inspect exact plan and state addresses

After applying only in a disposable test account:

```bash
terraform state list
terraform state show aws_s3_bucket.app
terraform output bucket_name
```

Expected Terraform address in each isolated state:

```text
aws_s3_bucket.app
```

Expected physical identifiers differ, for example:

| Environment | State key | Physical bucket |
|---|---|---|
| Dev | `application/dev/ap-south-1/terraform.tfstate` | `nikhil-app-dev-unique-12345` |
| QA | `application/qa/ap-south-1/terraform.tfstate` | `nikhil-app-qa-unique-12345` |
| Prod | `application/prod/ap-south-1/terraform.tfstate` | `nikhil-app-prod-unique-12345` |

The address is the logical Terraform identity; the bucket name is the physical AWS identity. Always record both when reviewing a plan.

To inspect which backend configuration a working directory has initialized, inspect its local Terraform metadata carefully and compare it with the intended backend file. Do not publish `.terraform` contents or state files because they can contain sensitive information.

## 8. Workspaces: know the difference

Terraform workspaces create multiple state instances within one configured backend.

Useful commands:

```bash
terraform workspace list
terraform workspace new dev
terraform workspace new qa
terraform workspace select dev
terraform workspace show
terraform state list
```

A workspace can be useful when environments are nearly identical and share the same access model. However:

- A workspace does not automatically select a different AWS account or role.
- A workspace is not an IAM/security boundary.
- A mistaken workspace selection can target the wrong environment.
- Workspaces can be less explicit for production approval and access separation.

Do not mix workspace-based isolation and manually separated backend keys without a deliberate design. Understand the resulting state path and workspace prefix.

## 9. Failure scenario — wrong backend key selected

Scenario: the QA pipeline is accidentally initialized with the dev backend file. The plan appears to contain unexpected changes because Terraform is reading dev state while using QA variables.

Safe recovery:

1. Stop. Do not apply the plan.
2. Record the intended environment, backend bucket/key, AWS account ID, and variable file.
3. Run `aws sts get-caller-identity` in the same execution context.
4. Inspect the initialized backend configuration and compare it with the approved QA backend file.
5. In a dedicated QA working directory, initialize against the correct backend configuration using `terraform init -reconfigure -backend-config=...` when the intent is to select the correct existing state without migrating the incorrectly selected state.
6. Do not use `-migrate-state` as a blind fix; it can copy the wrong environment's state into the intended backend.
7. Run `terraform state list` and `terraform state show aws_s3_bucket.app` to confirm the expected QA object is mapped.
8. Run a fresh plan with the QA variable file and inspect every create, update, and destroy action.
9. Apply only after the plan, state key, account ID, and approval all match QA.

If the wrong state was actually modified or copied, stop all operations and follow the team's state recovery procedure using S3 object versioning and a verified backup. Do not manually edit the state JSON.

## 10. Failure scenario — out-of-band drift in one environment

Scenario: someone changes the dev bucket's tags directly in AWS. QA and production should remain unaffected.

Tasks:

1. Initialize the dev working directory with the dev backend key.
2. Run `terraform plan` and identify the drift at `aws_s3_bucket.app`.
3. Confirm the live bucket name and AWS account before changing anything.
4. If Terraform configuration is the source of truth, restore the intended tag values through Terraform.
5. If another system owns that tag, agree on ownership and model it explicitly; do not add `ignore_changes` just to hide unexplained drift.
6. Run a fresh dev plan and confirm the correction is limited to dev.
7. Run QA and production plans separately and confirm they show no unexpected changes.

## 11. Production guardrails

- Use separate AWS accounts for production where possible.
- Use distinct deployment roles and least-privilege state-prefix permissions.
- Keep production credentials unavailable to ordinary dev/QA jobs.
- Require protected approvals before production plan/apply stages.
- Use separate CI working directories or jobs for each backend configuration.
- Verify AWS account identity and backend key before every production operation.
- Keep state bucket versioning, encryption, public-access blocks, TLS enforcement, and lockfiles enabled.
- Never commit state files, credentials, or sensitive backend metadata.
- Treat a saved plan as sensitive and ensure it is generated for the intended environment.

## 12. Interview-ready mental model

**Same reusable code, different backend key, different state, different credentials/account.** The state key isolates Terraform's record of resources; the AWS account and role isolate what the execution can actually change. A safe deployment verifies both before apply.
