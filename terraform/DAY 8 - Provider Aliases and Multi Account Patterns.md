# DAY 8 — Provider Aliases and Multi-Account Patterns

## 1. Why provider aliases matter

A provider configuration tells Terraform how to communicate with a platform. In AWS, it includes the region and the credentials or role Terraform uses.

A provider alias creates another configured instance of the same provider. This lets one root module manage resources in more than one AWS account or region.

Typical use cases:
- Separate development, test, and production AWS accounts.
- A central networking account and workload accounts.
- Deploying the same service to multiple regions.
- Reading identity or shared resources from one account while creating infrastructure in another.

**Critical distinction:** an alias is only a Terraform configuration label. It does not grant access to an account. The credentials or assumed role configured on that provider determine the actual AWS identity.

## 2. Configure two AWS provider instances

Use the AWS provider version constraint already established in the root module. Keep provider configuration in the root module.

Example `providers.tf`:

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
  alias  = "dev"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformExecutionRole"
  }
}

provider "aws" {
  alias  = "prod"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformExecutionRole"
  }
}
```

Replace the example account IDs and role names with approved values. Authenticate the source identity using your organization's approved method (for example, an AWS IAM Identity Center profile or workload identity). Do not put access keys or secrets in Terraform files.

The two aliases are:
- `aws.dev`
- `aws.prod`

Both use the same region here, but each assumes a role in a different account.

### How cross-account role assumption works

1. Terraform starts with a source AWS identity.
2. The target account contains an IAM role that trusts that source identity (or its approved identity provider).
3. The source identity has permission to call `sts:AssumeRole` on the target role.
4. The target role's permissions policy allows the required infrastructure operations.
5. The AWS provider uses the assumed-role credentials for its API calls.

Both sides matter: the trust policy controls who may assume the role; the role permissions control what the assumed identity may do. An external ID may be required by an organization's trust policy, especially for third-party access. Follow the account's security design rather than adding one automatically.

## 3. Verify the identity behind each alias

Never infer the target account from an alias name. Ask AWS STS which identity each provider actually uses.

```hcl
data "aws_caller_identity" "dev" {
  provider = aws.dev
}

data "aws_caller_identity" "prod" {
  provider = aws.prod
}

output "dev_account_id" {
  value = data.aws_caller_identity.dev.account_id
}

output "prod_account_id" {
  value = data.aws_caller_identity.prod.account_id
}
```

The data source addresses are:

```text
data.aws_caller_identity.dev
data.aws_caller_identity.prod
```

Compare the planned/output account IDs with the approved account inventory before applying. Do not expose sensitive identity or credential material; account IDs are identifiers, not credentials.

## 4. Select the provider on each resource

A resource uses the default provider configuration unless you specify an alias with its `provider` meta-argument.

```hcl
resource "aws_s3_bucket" "dev" {
  provider = aws.dev
  bucket   = "company-dev-example-unique-12345"
}

resource "aws_s3_bucket" "prod" {
  provider = aws.prod
  bucket   = "company-prod-example-unique-67890"
}
```

The exact Terraform resource addresses are:

```text
aws_s3_bucket.dev
aws_s3_bucket.prod
```

The address does not include the provider alias. Terraform records provider configuration association in state separately. The physical bucket names must be globally unique; use names approved for your test environment.

**Production safety rule:** make the provider choice visible in the resource or module call. Do not rely on an engineer remembering which credentials happen to be active in their shell.

## 5. Pass provider aliases into child modules

A child module should declare which provider configurations it can receive. It should not hardcode account credentials or create its own provider configuration.

Inside the child module, for example `modules/account_bucket/versions.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      configuration_aliases = [aws.target]
    }
  }
}
```

Inside `modules/account_bucket/main.tf`:

```hcl
resource "aws_s3_bucket" "this" {
  provider = aws.target
  bucket   = var.bucket_name
}
```

In the root module:

```hcl
module "dev_bucket" {
  source = "./modules/account_bucket"

  providers = {
    aws.target = aws.dev
  }

  bucket_name = "company-dev-module-example-unique-12345"
}

module "prod_bucket" {
  source = "./modules/account_bucket"

  providers = {
    aws.target = aws.prod
  }

  bucket_name = "company-prod-module-example-unique-67890"
}
```

The module call addresses and child resource addresses are:

```text
module.dev_bucket
module.dev_bucket.aws_s3_bucket.this
module.prod_bucket
module.prod_bucket.aws_s3_bucket.this
```

The `providers` map is an explicit handoff: the child module's `aws.target` configuration receives the root's selected provider instance. The child module does not independently choose an account.

For a simpler child module that uses only the default `aws` provider name internally, map it as `aws = aws.dev` or `aws = aws.prod`; declare `configuration_aliases` only when the child itself refers to a named alias such as `aws.target`.

## 6. Default provider traps

If a root module contains a normal unaliased provider block plus aliases, resources without a `provider` argument use the default provider. That can accidentally send a resource to the wrong account.

If every root provider configuration is aliased and no unaliased default is defined, Terraform may create an implied empty default configuration. A resource that accidentally omits its provider can then fail due to missing required configuration, rather than selecting one of the aliases.

Do not treat either behavior as a safety control. Explicitly set the provider for cross-account resources and modules, and verify account identity.

## 7. Commands: inspect provider selection and state

```bash
terraform fmt -recursive
terraform init
terraform providers
terraform validate
terraform plan
terraform plan -out=tfplan
terraform show tfplan
terraform state list
terraform state show 'aws_s3_bucket.dev'
terraform state show 'aws_s3_bucket.prod'
terraform state show 'module.dev_bucket.aws_s3_bucket.this'
terraform state show 'module.prod_bucket.aws_s3_bucket.this'
terraform output dev_account_id
terraform output prod_account_id
```

- `fmt -recursive`: format root and child-module HCL.
- `init`: initialize provider plugins and module dependencies.
- `providers`: show provider requirements and where configurations are used.
- `validate`: check configuration syntax and internal consistency.
- `plan`: preview changes without applying them.
- `plan -out=tfplan`: save the reviewed plan.
- `show tfplan`: inspect the saved plan.
- `state list`: list Terraform resource addresses.
- `state show <address>`: inspect one object tracked at an address.
- `output <name>`: display one root output.

A provider alias is not part of the resource address. Use `terraform providers`, the configuration, and state details to investigate provider association; use the account identity data sources to verify the actual AWS account.

## 8. Failure scenario — resource deployed to the wrong account

A developer adds a production S3 bucket but forgets `provider = aws.prod`. The resource is therefore planned against the default provider, which may be the development account.

Safe recovery:
1. Stop before apply. Do not assume the alias name guarantees the account.
2. Inspect the resource block and identify its effective provider configuration.
3. Inspect `terraform providers` and the `aws_caller_identity` data-source results.
4. Add the explicit `provider = aws.prod` (or correct the module `providers` mapping).
5. Run `terraform plan` and inspect the account identity and every proposed action.
6. If the resource was already applied in the wrong account, do not simply edit state or run `state rm`. First establish the physical resource, ownership, data, dependencies, and approved migration plan. Moving infrastructure between accounts is not an ordinary provider-alias edit.
7. Apply only after the destination account, resource actions, data migration, and rollback plan have been reviewed.

A provider configuration change can cause Terraform to interact with a different remote API context. Treat it as a potentially high-impact change.

## 9. Failure scenario — AssumeRole access denied

Terraform fails with an STS `AccessDenied` or cannot assume the target role.

Investigate:
- Is the source AWS identity the one you expect?
- Does its identity policy allow `sts:AssumeRole` on the exact target role ARN?
- Does the target role trust policy trust the source principal or approved identity provider?
- Is an external ID, session tag, MFA condition, or organization condition required?
- Does the assumed role have the required service permissions?
- Are the account ID and role ARN correct?
- Is the session expired or blocked by an SCP, permissions boundary, or explicit deny?

Fix the missing trust or permission at the correct policy boundary. Do not solve an access problem by granting broad administrator access. Re-run identity verification and plan after the correction.

## 10. Drift and account mismatch recovery

A resource exists in AWS, but the Terraform plan shows unexpected creation, deletion, or changes after provider configuration was modified.

1. Stop before applying.
2. Verify the account ID and region for every alias using `aws_caller_identity` and the provider configuration.
3. Inspect the exact Terraform address with `terraform state show`.
4. Compare configuration, state, and the live resource in the account that owns it.
5. Confirm whether the provider mapping changed, the resource was manually moved/recreated, or the wrong state/backend/workspace is being used.
6. Restore the approved provider mapping if the change was accidental.
7. If a real account migration is intended, plan it as a separate migration with data transfer, resource dependencies, ownership, and state-address strategy documented.
8. Re-run plan and confirm it targets the intended account and contains only approved actions.

Never delete state to make an account mismatch disappear. State is Terraform's record of which remote objects belong to which resource addresses.

## 11. Common mistakes

- Assuming `alias = "prod"` automatically selects the production account.
- Forgetting `provider = aws.prod` on a resource.
- Passing the wrong provider in a module's `providers` map.
- Defining a child module alias without declaring it in `configuration_aliases`.
- Putting provider credentials or provider configuration inside reusable child modules.
- Confusing role trust policy with the assumed role's permissions policy.
- Ignoring SCPs, permission boundaries, session conditions, or explicit denies.
- Applying before verifying the account ID and region.
- Treating a provider mapping change as a safe way to migrate existing resources between accounts.
- Assuming provider aliases appear in Terraform resource addresses.

## 12. Interview-ready mental model

**Provider configuration = identity + region + API settings. Alias = named provider instance. Resource/module provider mapping = explicit routing. STS identity check = proof of the account Terraform will access.**

Before applying multi-account Terraform, answer three questions:
1. Which provider configuration will this resource use?
2. Which AWS account and region does that provider actually resolve to?
3. Does that identity have only the permissions required for this deployment?
