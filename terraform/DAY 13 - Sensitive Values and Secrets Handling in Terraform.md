# DAY 13 — Sensitive Values and Secrets Handling in Terraform

## 1. Why secrets need special handling

Terraform may need passwords, API tokens, private keys, and database credentials. Secrets can leak through HCL, tfvars files, state, saved plan files, CI logs, shell tracing, debug output, and build artifacts.

Treat state and saved plans as sensitive artifacts. Remote state encryption and access controls reduce exposure, but do not make unrestricted access safe.

## 2. Sensitive is not the same as ephemeral

| Mechanism | CLI display | State and plan persistence | Purpose |
|---|---|---|---|
| sensitive = true | Redacts normal output | Value can still be stored | Hide accidental display |
| ephemeral = true | Can be combined with sensitive | Value omitted from state and plan | Runtime-only value |
| Write-only resource argument | Provider-specific redaction | Value omitted from state and plan | Send a value without reading it back |

Version requirements:
- Terraform 0.15+: sensitive variables and outputs.
- Terraform 1.10+: ephemeral variables, child-module outputs, and ephemeral resources.
- Terraform 1.11+: provider-supported write-only managed-resource arguments.

A provider must explicitly support a write-only argument. Do not assume every resource has one.

## 3. Small AWS example — SSM SecureString without persisting its value

This example requires Terraform 1.11+ and an AWS provider version supporting aws_ssm_parameter.value_wo. The value comes from a CI secret store or secure runtime environment, not from HCL or a committed tfvars file.

~~~hcl
terraform {
  required_version = ">= 1.11.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0, < 7.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

variable "app_api_token" {
  description = "Application token supplied securely at runtime"
  type        = string
  sensitive   = true
  ephemeral   = true
}

variable "secret_revision" {
  description = "Increment when the secret value is rotated"
  type        = number
  default     = 1
}

resource "aws_ssm_parameter" "app_api_token" {
  name        = "/myapp/prod/api-token"
  description = "Runtime API token for myapp"
  type        = "SecureString"

  value_wo         = var.app_api_token
  value_wo_version = var.secret_revision

  tags = {
    Application = "myapp"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

output "secret_parameter_arn" {
  description = "ARN of the SSM parameter; does not expose its value"
  value       = aws_ssm_parameter.app_api_token.arn
}
~~~

What each setting does:
- sensitive redacts the variable from normal Terraform output.
- ephemeral prevents the input value from being persisted in Terraform state and plan files.
- value_wo is the AWS provider's write-only argument. The value is sent to SSM but is not written into Terraform state.
- value_wo_version is a non-secret revision number. Increment it when rotating the value so Terraform knows to send the new write-only value.
- The output exposes only the parameter ARN, not the secret.

Exact Terraform address:

~~~text
aws_ssm_parameter.app_api_token
~~~

Physical SSM parameter name:

~~~text
/myapp/prod/api-token
~~~

The ARN and parameter metadata may appear in state. The secret value should not, when this write-only path is used correctly.

## 4. Supply the value securely

For local practice, export a temporary environment variable from a secure terminal session:

~~~bash
export TF_VAR_app_api_token='replace-with-a-test-secret'
terraform plan
terraform apply
unset TF_VAR_app_api_token
~~~

Use only a fake test value in this lab. Avoid typing secrets into shared terminals, shell history, screenshots, chat, or command-line arguments. CI log masking helps but is not a substitute for avoiding output.

In CI/CD:
1. Store the secret in the CI platform's protected secret store or a dedicated secrets manager.
2. Inject it only into the Terraform job that needs it.
3. Set the environment variable expected by Terraform, such as TF_VAR_app_api_token.
4. Disable shell tracing around secret handling; never print the environment.
5. Restrict who can read job logs, artifacts, state, and plan files.
6. Rotate the secret if it may have been exposed.

## 5. Inspect plan and state safely

~~~bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply
terraform state list
terraform state show aws_ssm_parameter.app_api_token
terraform output secret_parameter_arn
~~~

Expected tracked address: aws_ssm_parameter.app_api_token.

Expected inspection:
- State contains the parameter name, ARN, type, version, tags, and other provider metadata.
- The write-only secret value is not stored in state.
- The output contains only the parameter ARN.
- The secret is not available through terraform output because no output exposes it.

Do not use terraform state pull, terraform show -json, or saved-plan inspection on production state in a shared terminal just to prove a secret is hidden. If validating a disposable lab, protect the files and inspect only what is necessary. Never paste state or plan contents into tickets or chat.

## 6. Failure scenario — secret rotation does not update the parameter

Scenario: the secret changes in the CI secret store, but secret_revision remains 1.

Terraform cannot read or compare a write-only value. The version argument is the change signal.

Safe recovery:
1. Confirm the CI job receives the intended new secret from the approved secret store.
2. Do not print the value to verify it.
3. Increment secret_revision from 1 to 2.
4. Run terraform plan and inspect the parameter address and proposed update.
5. Apply through the approved pipeline.
6. Verify the SSM parameter version or metadata without retrieving or displaying the decrypted value.
7. Confirm a fresh plan is stable.

Example revision change:

~~~hcl
variable "secret_revision" {
  description = "Increment when the secret value is rotated"
  type        = number
  default     = 2
}
~~~

The revision is not the secret. The pipeline must inject the matching new value and increment the revision in the same controlled change.

## 7. Common trap — sensitive variable with a normal resource argument

This is masked in normal output, but the value can still persist in state and plan files:

~~~hcl
variable "database_password" {
  type      = string
  sensitive = true
}

resource "example_database" "app" {
  password = var.database_password
}
~~~

The sensitive flag controls display; it does not guarantee non-persistence. If the provider offers a supported write-only argument, prefer that with an ephemeral input. If it does not, assume the value may be stored in state and secure state accordingly.

An AWS SSM SecureString does not automatically protect Terraform state. Using the ordinary value argument can persist plaintext in raw state even though AWS encrypts the parameter at rest. Prefer value_wo where supported.

## 8. Other secret-handling patterns

### Read an existing secret from Secrets Manager

A normal data source such as aws_secretsmanager_secret_version can return the decrypted secret into Terraform. Marking the result sensitive hides it from ordinary output, but the value can still be stored in state.

Where supported by the AWS provider and Terraform version, use the provider's ephemeral Secrets Manager secret-version resource for runtime-only access. Check exact provider documentation and allowed ephemeral contexts.

For application runtime secrets, it is often better to give the workload an IAM role and let the application retrieve the secret directly from Secrets Manager or Parameter Store. This avoids routing the secret through Terraform.

### Backend and CI security

- Use remote state with encryption in transit and at rest.
- Restrict state access to the smallest set of roles and CI jobs.
- Enable state versioning and audit access.
- Treat saved plan files as secrets; limit retention and artifact access.
- Never commit state, plan files, tfvars containing secrets, or local secret files.
- Add .terraform/, *.tfstate, *.tfstate.*, *.tfplan, *.plan, and secret variable files to .gitignore; review repository history if a secret was committed.
- Use short-lived workload identity or OIDC credentials rather than long-lived AWS access keys where possible.
- Do not enable verbose Terraform/provider debug logs in normal production runs.

## 9. Recovery if a secret was exposed

1. Treat it as compromised. Revoke or rotate it immediately; deleting the visible line is not enough.
2. Identify where it appeared: Git history, forks, CI logs, artifacts, local state, remote state versions, caches, and backups.
3. Restrict access and follow the organization's incident process.
4. Remove the secret from source control history using the approved repository procedure, understanding clones and caches may still contain it.
5. Securely replace or expire affected state/plan artifacts according to backend and retention procedures. Do not manually edit state JSON.
6. Review audit logs for unexpected use.
7. Migrate to an ephemeral/write-only or application-runtime retrieval pattern where supported.
8. Verify the old credential no longer works and the replacement is functioning.

## 10. Commands and checks

| Command | Purpose |
|---|---|
| terraform fmt | Formats configuration |
| terraform init | Initializes providers and backend |
| terraform validate | Checks configuration syntax and types |
| terraform plan | Reviews intended changes without applying |
| terraform state list | Lists tracked resource addresses |
| terraform state show aws_ssm_parameter.app_api_token | Inspects tracked metadata for the parameter |
| git status --short | Checks for accidental local files |
| git check-ignore -v <file> | Shows which ignore rule applies to a file |

Never put the actual secret in a command argument, source file, terminal transcript, or Git commit.

## 11. Production checklist

- [ ] No secret hardcoded in HCL, locals, outputs, or committed tfvars.
- [ ] Input is injected from an approved secret store or CI secret store.
- [ ] sensitive is used for redaction where appropriate.
- [ ] ephemeral is used only in supported Terraform contexts.
- [ ] Provider write-only argument is supported and used where available.
- [ ] Rotation/version mechanism is understood and tested.
- [ ] State and plan files are encrypted, access-controlled, and retained appropriately.
- [ ] CI logs, shell tracing, and artifacts cannot expose the secret.
- [ ] Runtime workloads retrieve application secrets directly when feasible.
- [ ] Exposed credentials are rotated, not merely deleted from a file.

## 12. Interview-ready mental model

**Sensitive = hide the display. Ephemeral = do not persist the value. Write-only = pass the value to a resource without reading it back.**

Always check Terraform version, provider support, and the exact resource argument. If the value reaches a normal resource argument or data source, assume it may be stored in state unless the documentation explicitly says otherwise.
