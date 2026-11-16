# DAY 16 — Terraform Testing Concepts and Policy Checks

## 1. Why test Terraform

A successful terraform validate proves that the configuration is structurally valid. It does not prove that the module behaves as intended, that a plan contains only permitted changes, or that organizational security rules are satisfied.

Use multiple layers:

| Layer | Typical tool | Question answered |
|---|---|---|
| Formatting and static checks | terraform fmt, validate, TFLint | Is the code formatted and structurally sound? |
| Module behavior tests | terraform test | Does the module produce the expected plan or infrastructure behavior? |
| Plan policy checks | OPA/Conftest, Sentinel | Does the proposed change follow organizational rules? |
| Security/configuration scanning | Checkov, tfsec (where maintained/appropriate) | Are known risky configuration patterns present? |
| Runtime verification | Cloud APIs, integration tests | Does the deployed resource behave correctly? |

No single tool replaces the others. A policy check is not a substitute for a module test, and a module test is not proof that a live production deployment is safe.

## 2. Terraform native test framework

Terraform's native test framework uses files ending in .tftest.hcl. It became generally available in Terraform 1.6. Tests can contain run blocks with plan or apply command modes.

A plan-mode test evaluates the planned configuration without creating the managed infrastructure. An apply-mode test creates real resources and must be treated as an integration test with real cost, permissions and cleanup requirements.

Example module under test:

~~~hcl
# main.tf
variable "bucket_name" {
  type = string
}

resource "aws_s3_bucket" "app" {
  bucket = var.bucket_name

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "app" {
  bucket = aws_s3_bucket.app.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

output "bucket_id" {
  value = aws_s3_bucket.app.id
}
~~~

Exact resource addresses:

~~~text
aws_s3_bucket.app
aws_s3_bucket_public_access_block.app
~~~

Example test file:

~~~hcl
# tests/bucket.tftest.hcl
run "bucket_has_expected_name_and_tags" {
  command = plan

  variables {
    bucket_name = "example-test-bucket-12345"
  }

  assert {
    condition     = aws_s3_bucket.app.bucket == "example-test-bucket-12345"
    error_message = "The bucket name must match the test input."
  }

  assert {
    condition     = aws_s3_bucket.app.tags["Environment"] == "dev"
    error_message = "The bucket must have the dev environment tag."
  }

  assert {
    condition     = aws_s3_bucket_public_access_block.app.block_public_acls
    error_message = "Public ACLs must be blocked."
  }
}
~~~

This is a plan-mode test. It checks planned values and does not create the bucket. Run it from the root module:

~~~bash
terraform init
terraform test
~~~

The AWS provider must be available and configured sufficiently for the selected test mode. Plan tests can still require provider schema initialization and may require credentials depending on provider/data-source behavior. Never assume every test is fully offline.

## 3. Test expected failures and input validation

A module should reject invalid input clearly rather than allowing an unsafe or confusing plan.

For example, add a validation rule:

~~~hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "qa", "prod"], var.environment)
    error_message = "environment must be dev, qa, or prod."
  }
}
~~~

A native test can assert that an invalid input fails:

~~~hcl
run "rejects_unknown_environment" {
  command = plan

  variables {
    bucket_name = "example-test-bucket-12345"
    environment = "sandbox"
  }

  expect_failures = [
    var.environment
  ]
}
~~~

For this example, the resource tags should use var.environment and the variable must be declared in the module. The test's purpose is to verify the validation rule rejects an unsupported value.

Run:

~~~bash
terraform test
~~~

Read the full test diagnostic. A test failure may indicate either that the module is wrong or that the test expectation itself is wrong.

## 4. Mock providers and safe testing

Terraform 1.7 introduced provider mocking for native tests. Mock providers can generate computed values and allow many module tests to run without creating real cloud resources.

A mock provider can be declared in a test file:

~~~hcl
mock_provider "aws" {}

run "plans_bucket" {
  command = plan

  variables {
    bucket_name = "example-test-bucket-12345"
  }

  assert {
    condition     = aws_s3_bucket.app.bucket == "example-test-bucket-12345"
    error_message = "The planned bucket name is incorrect."
  }
}
~~~

Mocking is useful for module logic and plan assertions. It does not prove that AWS accepts the request, the caller has IAM permissions, quotas are available, or the live service behaves as expected.

Use a separate disposable sandbox for integration tests that need real provider operations. Clean up resources and inspect state after apply-mode tests.

## 5. Policy as code: validate the plan against rules

A plan can be valid and still violate security or cost policies. Examples:

- S3 buckets must block public access.
- Production resources must have required ownership and environment tags.
- Encryption must be enabled.
- Only approved regions may be used.
- Production changes must not destroy protected resources without an exception.

Policy as code evaluates machine-readable plan data and returns pass/fail decisions.

Common approaches:
- Sentinel: policy framework commonly used with HCP Terraform/Terraform Enterprise.
- Open Policy Agent (OPA) with Conftest: evaluate Terraform plan JSON using Rego policies.
- Security scanners: tools such as Checkov inspect Terraform configuration and related artifacts for known misconfigurations.

Choose a policy tool that fits the team's platform, enforcement model and support requirements.

## 6. Generate plan JSON for policy evaluation

Terraform can export a plan in JSON format:

~~~bash
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
~~~

The JSON representation can contain sensitive values. Protect it like state and saved plan artifacts: never commit it, never publish it publicly, restrict access and retention, and delete it securely after the job according to organizational policy.

Example Conftest command:

~~~bash
conftest test tfplan.json --policy policy/
~~~

A simple illustrative Rego policy for checking that managed S3 buckets have public access blocking configured:

~~~rego
package main

deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.after.block_public_acls != true
  msg := sprintf("S3 public ACL blocking must be enabled for %s", [resource.address])
}
~~~

This rule is intentionally small and illustrative. A production policy must account for the exact Terraform plan JSON schema, resource action types, unknown/null values, deletes, replacements, module addresses and provider-specific resource patterns. Test policies against representative plan fixtures before enforcing them.

The policy should also detect a missing public-access-block resource for a bucket; checking only resources that are present does not prove every bucket is protected.

## 7. Policy checks in a CI pipeline

A sensible pull-request flow:

~~~text
Checkout
  |
  v
Terraform fmt check
  |
  v
Terraform init and validate
  |
  v
Terraform test
  |
  v
Terraform plan
  |
  v
Export protected plan JSON
  |
  v
OPA / Sentinel / security policy checks
  |
  v
Review plan and policy results
  |
  v
Approval
  |
  v
Protected apply workflow
  |
  v
Post-deployment verification
~~~

Important controls:
1. Pin Terraform and scanner versions.
2. Run tests in an isolated working directory with controlled credentials.
3. Do not expose secrets to untrusted pull-request code.
4. Use OIDC or another short-lived identity mechanism for trusted cloud operations.
5. Protect plan and JSON artifacts because they may contain sensitive values.
6. Fail the pipeline when a mandatory policy returns a violation.
7. Document exception ownership, approval and expiry; do not silently bypass a failed policy.
8. Separate plan-time policy evaluation from runtime security monitoring.

## 8. Failure scenario — a policy rejects an otherwise valid plan

Scenario: terraform validate, terraform test and terraform plan all pass, but the policy check rejects an S3 bucket because the public-access-block resource is missing or a required tag is absent.

Safe recovery:
1. Do not bypass the policy or apply the plan.
2. Read the policy identifier, message and exact resource address.
3. Inspect the plan and source HCL to determine whether the control is genuinely missing or the policy has a false positive.
4. Add the required resource/configuration or correct the policy with a reviewed test.
5. Re-run fmt, validate, terraform test, plan and policy checks.
6. Confirm the exact addresses, including aws_s3_bucket.app and aws_s3_bucket_public_access_block.app.
7. Obtain approval and apply through the protected deployment path.
8. Verify the live AWS configuration after deployment.

## 9. Failure scenario — test passes but real apply fails

A mock-provider or plan-mode test can pass while a real AWS apply fails due to IAM permissions, service quotas, invalid account settings or provider/API behavior.

Recovery:
1. Identify whether the failure happened during plan or apply.
2. Capture the exact failed resource address and provider diagnostic.
3. Verify the active AWS identity, region and permissions.
4. Inspect Terraform state and the actual AWS resource; apply may have partially succeeded.
5. Fix the real integration issue.
6. Run a fresh plan and review the remaining actions.
7. Re-run the relevant test and policy gates before retrying.

## 10. Common mistakes

- Assuming terraform validate is a test suite.
- Assuming terraform test always creates real resources.
- Using apply-mode tests against production.
- Treating mocked provider success as proof of cloud API success.
- Exporting plan JSON to an unprotected artifact.
- Writing a policy that checks only resources that exist and misses missing mandatory resources.
- Ignoring unknown or null values in plan JSON.
- Writing a policy that accidentally allows destructive replacement actions.
- Silencing a policy failure instead of documenting a reviewed exception.
- Running policy checks against configuration only when the actual requirement concerns planned actions.
- Failing to test the policy itself with pass and fail fixtures.

## 11. Commands and exact addresses

~~~bash
terraform fmt -check -recursive
terraform init
terraform validate
terraform test
terraform plan -out=tfplan
terraform show tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policy/
terraform state list
terraform state show aws_s3_bucket.app
terraform state show aws_s3_bucket_public_access_block.app
~~~

Expected managed resource addresses:

~~~text
aws_s3_bucket.app
aws_s3_bucket_public_access_block.app
~~~

Use the exact addresses from your own plan/state if resource names differ. Protect tfplan and tfplan.json as sensitive artifacts.

## 12. Interview-ready mental model

**validate checks structure; test checks module behavior; policy checks enforce organizational rules on configuration or planned changes; integration tests verify real provider/cloud behavior.**

A robust Terraform pipeline tests the code, evaluates the proposed plan against mandatory controls, requires review, applies through a protected workflow and verifies the deployed result.
