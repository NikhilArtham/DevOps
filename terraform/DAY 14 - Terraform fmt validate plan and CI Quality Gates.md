# DAY 14 — Terraform fmt, validate, plan and CI Quality Gates

## 1. Why these checks exist

Terraform quality gates catch different classes of problems before infrastructure changes are approved:

- `terraform fmt` makes configuration formatting consistent.
- `terraform init` installs providers and initializes the backend/module dependencies.
- `terraform validate` checks whether the initialized configuration is structurally valid.
- `terraform plan` evaluates the proposed infrastructure changes.
- CI quality gates run these checks consistently for every pull request and protect the apply path.

A successful validation does not mean a plan is safe. A successful plan does not mean the change is approved. Each gate answers a different question.

## 2. Small configuration

Create a disposable directory named `terraform-quality-gates` with these files:

~~~text
terraform-quality-gates/
├── versions.tf
├── providers.tf
├── variables.tf
├── main.tf
└── outputs.tf
~~~

### versions.tf

~~~hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
~~~

### providers.tf

~~~hcl
provider "aws" {
  region = var.aws_region
}
~~~

### variables.tf

~~~hcl
variable "aws_region" {
  description = "AWS region for the test bucket"
  type        = string
  default     = "ap-south-1"
}

variable "bucket_name" {
  description = "Globally unique name for the test bucket"
  type        = string

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "bucket_name must contain between 3 and 63 characters."
  }
}
~~~

### main.tf

~~~hcl
resource "aws_s3_bucket" "app" {
  bucket = var.bucket_name

  tags = {
    Name        = "quality-gates-demo"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
~~~

### outputs.tf

~~~hcl
output "bucket_arn" {
  description = "ARN of the managed test bucket"
  value       = aws_s3_bucket.app.arn
}
~~~

The exact managed resource address is:

~~~text
aws_s3_bucket.app
~~~

The bucket name is the physical AWS identifier. Use a unique test name and a sandbox account; do not use production resources for failure exercises.

## 3. Run each quality check locally

Run commands from the configuration directory:

~~~bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -var="bucket_name=replace-with-a-unique-test-name"
~~~

What each command checks:

| Command | What it checks | What it does not prove |
|---|---|---|
| terraform fmt -check -recursive | Files already match canonical formatting | HCL is valid or safe |
| terraform init -backend=false | Downloads providers/modules without initializing the configured backend | Cloud credentials or remote access work |
| terraform validate | Syntax, references, types and internal configuration consistency | Real infrastructure changes are safe |
| terraform plan | Evaluates proposed actions using configuration, state and provider refresh | Change is approved or risk-free |

For a CI formatting gate, use check mode so CI reports formatting drift without silently modifying files:

~~~bash
terraform fmt -check -recursive
~~~

For local remediation, run `terraform fmt -recursive`, inspect the diff, and commit the formatted files.

## 4. Understand the plan before approving it

Run a plan with the intended variables:

~~~bash
terraform plan -out=tfplan -var="bucket_name=replace-with-a-unique-test-name"
terraform show tfplan
~~~

Expected address:

~~~text
aws_s3_bucket.app
~~~

Common plan symbols:

- `+`: create.
- `-`: destroy.
- `~`: update in place.
- `-/+`: destroy and then create replacement.
- `+/-`: create replacement before destroying the old object, when supported by lifecycle ordering.

A plan can be syntactically valid and still be operationally dangerous. Inspect resource addresses, before/after values, replacement markers, account, region, backend, workspace and provider alias.

Plan files can contain sensitive values. Store them as protected, short-lived CI artifacts; restrict download access and retention. Do not commit `tfplan`.

## 5. GitHub Actions CI quality gates

Example workflow for pull requests. This workflow performs formatting, initialization without backend access, and validation. It intentionally does not apply infrastructure.

~~~yaml
name: Terraform Quality Gates

on:
  pull_request:
    paths:
      - "terraform/**"
      - ".github/workflows/terraform-quality.yml"

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform-quality-gates

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.11.4

      - name: Check formatting
        run: terraform fmt -check -recursive

      - name: Initialize without backend
        run: terraform init -backend=false

      - name: Validate configuration
        run: terraform validate -no-color
~~~

Adapt the working directory and Terraform version to the repository's pinned version. Pin third-party GitHub Actions to reviewed immutable commit SHAs in a hardened production workflow; version tags are shown here for readability.

### Plan gate design

A real plan job may need backend access, provider credentials, state read/lock permissions, and remote API access. Design it separately:

1. Run only for trusted pull-request contexts where credentials can be safely issued.
2. Prefer OIDC/workload identity and short-lived role credentials; do not store long-lived AWS keys in repository files.
3. Give the plan role read/refresh permissions and only the state-lock permissions needed by the backend.
4. Do not run privileged plans for untrusted fork code with secrets.
5. Upload the plan only as a restricted, short-lived artifact if reviewers need it.
6. Require protected environment approval and a fresh, controlled apply workflow.
7. Never run automatic production apply from an ordinary pull-request validation job.

A saved plan can become stale if state or infrastructure changes after it is created. Re-plan and re-review when the approved workflow requires a fresh plan.

## 6. Failure scenario — formatting and validation fail in CI

Scenario A: a developer commits inconsistent indentation. CI fails at `terraform fmt -check -recursive`.

Recovery:
1. Run `terraform fmt -recursive` locally.
2. Inspect the formatting diff.
3. Commit the corrected HCL.
4. Push and confirm the formatting gate passes.

Scenario B: a resource references an undeclared variable or invalid attribute. Formatting passes, but validation fails.

Recovery:
1. Read the first meaningful Terraform diagnostic and file/line location.
2. Correct the variable declaration, resource reference, type, or provider schema issue.
3. Run init, validate and plan again.
4. Confirm the exact address `aws_s3_bucket.app` and intended changes.

Scenario C: validation passes but plan proposes an unexpected replacement.

Recovery:
1. Stop; do not apply.
2. Inspect `terraform show tfplan` or generate a fresh plan.
3. Identify the exact resource address and attribute causing replacement.
4. Verify AWS identity, region, backend key, workspace and provider alias.
5. Decide whether the change is intended; correct HCL or escalate for review.
6. Re-run all gates and obtain approval for the final plan.

## 7. Common mistakes

- Treating fmt, validate and plan as interchangeable checks.
- Running validate before init when providers/modules are not installed.
- Assuming `terraform init -backend=false` verifies remote backend access.
- Letting CI run `terraform fmt` and silently modify the pull-request workspace instead of failing with fmt check.
- Running plans with credentials from untrusted fork pull requests.
- Printing secrets or full environment variables in CI logs.
- Uploading plan files as public or long-retention artifacts.
- Automatically applying a pull-request plan to production.
- Ignoring replacement actions because validation passed.
- Using a different Terraform/provider version locally and in CI.

## 8. Troubleshooting checklist

- Does the CI job use the expected Terraform CLI version?
- Is the working directory correct?
- Does fmt check identify a file that needs formatting?
- Did init install the expected providers and modules?
- Is the lock file committed and unchanged unexpectedly?
- Does validate point to an undeclared variable, wrong type or invalid reference?
- Does plan use the intended backend, workspace, account, region and provider alias?
- Which exact resource address is changing?
- Is the action create, update, destroy or replacement?
- Are credentials and plan artifacts protected?
- Is apply separated behind the required approval?

## 9. Interview-ready mental model

**fmt = style; init = dependencies/backend setup; validate = configuration correctness; plan = proposed infrastructure actions; CI gates = repeatable enforcement before approval.**

Passing earlier gates never substitutes for reviewing the plan.