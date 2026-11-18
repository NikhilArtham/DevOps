# Terraform Interview Questions

## DAY 1 — Professional Workflow, Providers and Versions

### 1. Why should Terraform and provider versions be constrained?

<details><summary>Answer</summary>

Version constraints make Terraform execution reproducible and prevent unexpected dependency upgrades. Terraform CLI and provider constraints define what versions are acceptable, while `.terraform.lock.hcl` records the selected provider version and checksums for consistent initialization.

</details>

### 2. What is the difference between a Terraform provider and Terraform itself?

<details><summary>Answer</summary>

Terraform is the infrastructure-as-code engine that evaluates configuration, builds plans, manages state, and coordinates changes. A provider is a plugin that implements communication with an external platform such as AWS, Azure, Kubernetes, or GitHub.

</details>

### 3. What does `~> 6.0` mean for a provider version?

<details><summary>Answer</summary>

It allows compatible releases in the 6.x series while excluding 7.x. It is commonly used when you want controlled minor/patch upgrades without allowing a major-version change.

</details>

### 4. What is `.terraform.lock.hcl`?

<details><summary>Answer</summary>

It is Terraform's dependency lock file. It records selected provider versions and checksums so different developers and CI environments can initialize the same provider dependency consistently. It should normally be committed to version control.

</details>

### 5. Does Terraform execute `.tf` files in filename order?

<details><summary>Answer</summary>

No. Terraform loads all `.tf` files in a root module and evaluates the configuration as a whole. Splitting files such as `providers.tf`, `variables.tf`, and `main.tf` is primarily for organization and maintainability.

</details>

### 6. What is a Terraform resource address?

<details><summary>Answer</summary>

A resource address identifies a specific resource instance in Terraform. For example, `aws_s3_bucket.example` identifies the `example` resource of type `aws_s3_bucket`. Resource addresses are important for state inspection, imports, targeted operations, and troubleshooting plans.

</details>

### 7. How would you investigate an unexpected Terraform plan?

<details><summary>Answer</summary>

First identify the exact resource address and whether Terraform wants to create, update, replace, or destroy it. Compare the configuration, state, and real infrastructure. Check provider and Terraform versions, the lock file, and recent configuration changes. Use `terraform state show <address>` and provider/version inspection commands before deciding on a remediation.

</details>

### 8. What would you do if a provider upgrade caused unexpected changes?

<details><summary>Answer</summary>

Stop before applying. Inspect the provider constraint and lock-file change, compare the old and new provider versions, and review the plan. If the upgrade was not intentional, restore the approved lock file and reinitialize. If it was intentional, test and review the resulting plan before approving the change.

</details>

### 9. Should you delete Terraform state when there is drift?

<details><summary>Answer</summary>

Normally no. State is Terraform's mapping between resource addresses and real infrastructure. Drift should be investigated by comparing configuration, state, and real infrastructure. State manipulation is a specialized recovery operation, not a general solution for plan differences.

</details>

## DAY 2 — Variables, Locals, Outputs and Type Constraints

### 10. What is a Terraform variable?

<details><summary>Answer</summary>

A variable is an input to a module. It lets callers provide values without hard-coding environment-specific configuration inside the module.

</details>

### 11. What is the difference between a variable and a local?

<details><summary>Answer</summary>

A variable is supplied to a module from outside. A local is an internally calculated or reusable value. Variables define the module's input interface; locals help organize internal expressions.

</details>

### 12. Why should you use type constraints?

<details><summary>Answer</summary>

Type constraints define the expected shape of module inputs, such as `string`, `number`, `list(string)`, or an object. They catch invalid inputs early and make modules easier to reuse safely.

</details>

### 13. What is the difference between a type constraint and variable validation?

<details><summary>Answer</summary>

The type constraint defines what kind of value is accepted. Validation applies additional rules to the value. For example, `type = string` allows strings, while validation can restrict the string to `dev`, `test`, or `prod`.

</details>

### 14. What is the difference between a list, set, and map in Terraform?

<details><summary>Answer</summary>

A list is ordered and can contain repeated values. A set is an unordered collection of unique values. A map stores key-value pairs, commonly with a consistent value type such as `map(string)`.

</details>

### 15. How do outputs work between child and root modules?

<details><summary>Answer</summary>

A child module declares an output, and the parent accesses it with `module.<module_name>.<output_name>`. If the value must appear in `terraform output` for the root module, the root module must declare its own output that references the child output.

</details>

### 16. Do variables, locals, or outputs create AWS resources?

<details><summary>Answer</summary>

No. They are Terraform language constructs. Resources such as `aws_s3_bucket` create or manage AWS infrastructure. Variables provide values, locals calculate values, and outputs expose selected results.

</details>

### 17. What happens if a number variable receives a string such as `"three"`?

<details><summary>Answer</summary>

Terraform reports a type error because the value does not satisfy `type = number`. The correct response is to fix the input at its source rather than weakening the module's type constraint just to make the plan succeed.

</details>

### 18. Does `sensitive = true` on an output remove the secret from Terraform state?

<details><summary>Answer</summary>

No. It mainly prevents Terraform from displaying the value normally in CLI output and plans. Sensitive values can still exist in state, so state storage and access must be secured.

</details>

### 19. Which commands are useful for troubleshooting variables and locals?

<details><summary>Answer</summary>

Use `terraform validate` for configuration checks, `terraform plan` to evaluate the resulting changes, `terraform console` to test expressions and types, and `terraform output` to inspect root outputs after values are stored in state.

</details>

### 20. How would you troubleshoot a production variable failure?

<details><summary>Answer</summary>

First identify the variable and its declared type and validation rules. Then trace where its value came from, such as `-var`, `.tfvars`, or `TF_VAR_*`. Check any local expressions using the value, identify the exact resource address affected, and review the plan before applying. Fix the input at its source rather than weakening the module contract.

</details>

## DAY 3 — Resource Lifecycle, Dependencies and Replace Behavior

### 21. What does `-/+` mean in a Terraform plan?

<details><summary>Answer</summary>

It means Terraform must replace the resource: destroy the existing instance and create a new one. The replacement is usually required because a changed attribute cannot be updated in place by the resource/provider.

</details>

### 22. What is `create_before_destroy`?

<details><summary>Answer</summary>

It is a lifecycle setting that asks Terraform to create a replacement before destroying the existing resource when possible. It can reduce downtime, but it does not guarantee zero downtime because provider constraints, uniqueness, dependencies, and quotas can prevent parallel existence.

</details>

### 23. What is `prevent_destroy` used for?

<details><summary>Answer</summary>

It is a lifecycle guardrail that causes Terraform to reject a plan/apply that would destroy the resource. It can protect critical resources, but it may also block legitimate replacement or deletion operations.

</details>

### 24. What is `ignore_changes` and why can it be dangerous?

<details><summary>Answer</summary>

`ignore_changes` tells Terraform to ignore changes to specified attributes during planning. It can be appropriate when another system intentionally owns an attribute, but it can also hide drift and make Terraform's configuration differ from reality without obvious plan output.

</details>

### 25. What is the difference between implicit and explicit dependencies?

<details><summary>Answer</summary>

An implicit dependency is created automatically when one resource references another resource's value. An explicit dependency uses `depends_on` when Terraform cannot infer an operational dependency from configuration.

</details>

### 26. Should `depends_on` be added to every resource relationship?

<details><summary>Answer</summary>

No. Prefer direct resource references because they express the actual relationship and allow Terraform to build a more precise graph. Use `depends_on` only when a real dependency exists but cannot be inferred from an expression.

</details>

### 27. How do you troubleshoot an unexpected replacement?

<details><summary>Answer</summary>

Stop before applying. Identify the exact resource address, inspect the plan for the changed attribute, determine why the provider requires replacement, and verify whether the change is intentional. Check dependencies, capacity, uniqueness constraints, persistence, and whether `create_before_destroy` is safe before approving the replacement.

</details>

### 28. How can you inspect Terraform's dependency graph?

<details><summary>Answer</summary>

Run `terraform graph`. The command produces a graph representation of Terraform's dependency relationships. It is useful when troubleshooting resource ordering or validating whether an explicit dependency is actually needed.

</details>

### 29. Does `create_before_destroy` always prevent downtime?

<details><summary>Answer</summary>

No. It only changes the preferred replacement ordering. Resource naming rules, uniqueness constraints, quotas, provider behavior, health checks, load balancing, and application architecture can still cause downtime or prevent parallel creation.

</details>

### 30. Should you delete Terraform state to fix an unexpected replacement?

<details><summary>Answer</summary>

Normally no. First identify the changed attribute and compare configuration, state, and real infrastructure. State deletion can make Terraform lose its mapping to existing infrastructure and is not a normal replacement-recovery technique.

</details>


## DAY 4 — `count` vs `for_each` and Stable Resource Addressing

### 31. What is the difference between `count` and `for_each`?

<details><summary>Answer</summary>

`count` creates indexed instances such as `aws_instance.app[0]`. `for_each` creates keyed instances such as `aws_instance.app["api"]`. Use `count` for simple positional or identical instances; use `for_each` when each instance has a meaningful identity that should remain stable as the collection changes.

</details>

### 32. What is a Terraform resource address?

<details><summary>Answer</summary>

A resource address is Terraform's identifier for a resource instance in configuration and state. Examples are `aws_instance.app`, `aws_instance.app[0]`, and `aws_instance.service["api"]`. It is not the physical AWS resource ID.

</details>

### 33. Why can removing an item from the middle of a `count` list be risky?

<details><summary>Answer</summary>

`count` uses numeric positions as instance identity. Removing a middle list item shifts the values at later indexes. Terraform may then update or replace the object at an index and destroy the final index, even though the developer intended to remove only one logical item.

</details>

### 34. Why is `for_each` often safer for named infrastructure?

<details><summary>Answer</summary>

Each instance is identified by a stable map key or set member, such as `"api"` or `"worker"`. Removing one key does not shift the identity of the other instances. This makes the plan easier to reason about when named resources are added or removed independently.

</details>

### 35. What are `each.key` and `each.value`?

<details><summary>Answer</summary>

Inside a resource using `for_each`, `each.key` is the current map key or set member. `each.value` is the corresponding map value. For a set of strings, the key and value are the same string. For a map, the value can hold per-instance configuration.

</details>

### 36. Can a resource use both `count` and `for_each`?

<details><summary>Answer</summary>

No. A single resource or module block can use either `count` or `for_each`, not both. Choose the one that best represents the identity of the instances.

</details>

### 37. What happens to resource addresses when converting from `count` to `for_each`?

<details><summary>Answer</summary>

The addresses change from numeric indexes, such as `aws_instance.app[0]`, to keyed addresses, such as `aws_instance.service["api"]`. Without an explicit migration, Terraform may interpret the new address as a new object and the old address as one to destroy. Use correctly mapped `moved` blocks or a carefully managed `terraform state mv` operation, then verify the plan.

</details>

### 38. What is a `moved` block?

<details><summary>Answer</summary>

A `moved` block declares that a resource instance previously tracked at one address should now be tracked at another address. It lets Terraform update its state address mapping without treating the move itself as destruction and recreation. The mapping must match the actual existing instance.

</details>

### 39. How do you inspect instances created with `for_each`?

<details><summary>Answer</summary>

Run `terraform state list` to see all tracked addresses. Then inspect a specific instance with a quoted address, for example `terraform state show 'aws_instance.service["api"]'`. Also inspect `terraform plan` to understand proposed changes before applying.

</details>

### 40. When is `count` still a good choice?

<details><summary>Answer</summary>

It is suitable for a simple number of interchangeable instances where positional identity is acceptable, or for a conditional singleton using `count = condition ? 1 : 0`. If instances are independently named or may be selectively removed or reordered, `for_each` is usually a better fit.

</details>


## DAY 5 — Dynamic Blocks, Expressions and Conditional Infrastructure

### 41. What is a Terraform expression?

<details><summary>Answer</summary>

An expression represents a value. It can reference variables, locals, resources, data sources, operators, and functions. Expressions are used to calculate configuration values and transform collections.

</details>

### 42. What is a conditional expression?

<details><summary>Answer</summary>

A conditional expression uses the form `condition ? true_value : false_value`. Terraform evaluates the condition and returns one of the two values. It is useful for environment-specific settings and optional values.

</details>

### 43. What is a Terraform `for` expression?

<details><summary>Answer</summary>

A `for` expression iterates over a list, set, or map and constructs a new collection. It can also filter elements using an `if` clause. It is useful for transforming tags, names, and configuration data.

</details>

### 44. What is a dynamic block?

<details><summary>Answer</summary>

A dynamic block generates repeated nested configuration blocks from a collection. It is useful when a resource supports repeatable nested blocks, such as ingress rules inside an AWS security group. It does not create separate Terraform resource instances.

</details>

### 45. What is the difference between dynamic blocks and resource-level `for_each`?

<details><summary>Answer</summary>

Resource-level `for_each` creates multiple independently addressed resource instances. A dynamic block generates multiple nested blocks inside one resource instance. For example, resource-level `for_each` can create several security groups, while a dynamic ingress block can generate several ingress rules inside one security group.

</details>

### 46. What are `for_each` and `content` used for inside a dynamic block?

<details><summary>Answer</summary>

The dynamic block's `for_each` defines the collection to iterate over. The `content` block describes the nested block generated for each element. The iterator exposes the current element's key and value.

</details>

### 47. Can dynamic blocks generate any Terraform block?

<details><summary>Answer</summary>

No. A dynamic block can generate only nested block types supported by the parent resource or data source schema. It cannot generate meta-argument blocks such as `lifecycle` or `provisioner`, because Terraform needs to process those earlier.

</details>

### 48. How do you conditionally create a resource?

<details><summary>Answer</summary>

For an optional singleton, use `count = condition ? 1 : 0`. For multiple named optional resources, use `for_each` with a map or set. Remember that changing count from one to zero plans to destroy the previously managed resource.

</details>

### 49. How would you troubleshoot a dynamic block error?

<details><summary>Answer</summary>

Check that the dynamic block label is supported by the provider resource schema, verify the input collection's type and object keys, and confirm the iterator references are correct. Then run `terraform fmt`, `terraform validate`, and `terraform plan`. Inspect the generated nested configuration and do not apply until the plan is correct.

</details>

### 50. Why should you validate inputs used to generate security-group rules?

<details><summary>Answer</summary>

A structurally valid rule can still be insecure. Validate port ranges, protocols, and approved CIDR ranges so a reusable module does not accidentally expose management or application ports to untrusted networks. Review the generated plan before applying.

</details>



## DAY 6 — Reusable Terraform Module Design

### 51. What is a Terraform module?

<details><summary>Answer</summary>

A Terraform module is a collection of Terraform configuration files in one directory. The root configuration is the root module, and a module called by another module is a child module. Modules package resources behind inputs and outputs so infrastructure patterns can be reused.

</details>

### 52. What is the difference between a root module and a child module?

<details><summary>Answer</summary>

The root module is the working directory where Terraform is run. It commonly configures providers, supplies environment-specific inputs, calls child modules, and owns the deployment state. A child module encapsulates a capability using its own variables, resources, locals, and outputs. A child module normally shares the root module's state.

</details>

### 53. Why should provider configuration normally live in the root module?

<details><summary>Answer</summary>

The root module is the deployment entry point and knows the target account, region, credentials, and provider aliases. Keeping provider configuration there allows the same child module to be reused across environments and accounts. Child modules can declare provider requirements and receive provider configurations from callers without embedding credentials.

</details>

### 54. What makes a Terraform module reusable?

<details><summary>Answer</summary>

A reusable module has a clear and stable input contract, typed variables, meaningful validation, sensible defaults, useful outputs, minimal hardcoding, and documentation. It should encapsulate a real infrastructure capability without exposing unnecessary complexity or adding abstraction without value.

</details>

### 55. What is the purpose of module inputs and outputs?

<details><summary>Answer</summary>

Inputs let callers configure a module without editing its internal resources. Outputs return useful values such as IDs, ARNs, endpoints, or names to the caller. A caller references a child output as `module.<module_call_name>.<output_name>`.

</details>

### 56. How do you reference a resource created inside a child module?

<details><summary>Answer</summary>

A resource address includes the module call name. For example, a resource declared as `aws_s3_bucket.this` inside a child module called `data_bucket` is addressed as `module.data_bucket.aws_s3_bucket.this`. This Terraform address is different from the physical AWS resource ID.

</details>

### 57. Does a child module have a separate state file by default?

<details><summary>Answer</summary>

No. A child module normally uses the state of the root configuration that calls it. Its resources are tracked using module-qualified addresses. A separate root configuration with its own backend/state is a separate deployment boundary.

</details>

### 58. How can you use the same module more than once?

<details><summary>Answer</summary>

Call the same module source from multiple module blocks with different call names and input values. For example, `module "data_bucket"` and `module "logs_bucket"` can both use `./modules/s3_bucket`. Their resources receive different addresses because the module call names differ.

</details>

### 59. What happens if you rename a module block?

<details><summary>Answer</summary>

Renaming a module call changes the resource addresses in state. Terraform may interpret the new addresses as new resources and the old addresses as resources to destroy. Use a correctly scoped `moved` block or a controlled state migration, then inspect the plan to confirm Terraform recognizes the address move instead of replacing infrastructure.

</details>

### 60. How do you troubleshoot invalid module input or drift?

<details><summary>Answer</summary>

For invalid input, inspect the caller's value, child variable type, and validation condition; correct the value at its source and re-run validation and plan. For drift, identify the exact module-qualified address, compare configuration, state, and live infrastructure, decide which system owns the attribute, and review the smallest safe correction. Do not delete state or suppress unexplained drift with `ignore_changes`.

</details>


## DAY 7 — Module Inputs, Outputs, Validation and Documentation

### 61. What makes a Terraform module input contract good?

<details><summary>Answer</summary>

A good contract has explicit types, clear descriptions, required values without misleading defaults, safe optional defaults, and validation for important rules. It exposes decisions callers need to make without leaking internal implementation details.

</details>

### 62. What is the difference between a type constraint and variable validation?

<details><summary>Answer</summary>

A type constraint defines the shape or data type Terraform accepts, such as `string`, `number`, or `list(string)`. Variable validation adds rules about acceptable values, such as allowing only `dev`, `test`, or `prod`.

</details>

### 63. When should a variable have a default?

<details><summary>Answer</summary>

Use a default when there is a safe and predictable value. Keep important environment-specific values required. A default should not hide a decision callers must consciously make, such as target account or network selection.

</details>

### 64. What does `nullable = false` do?

<details><summary>Answer</summary>

It prevents a variable from receiving null as its value. Use it when null is not meaningful. It does not replace type constraints or validation rules.

</details>

### 65. What is the difference between variable validation, preconditions, and postconditions?

<details><summary>Answer</summary>

Variable validation checks caller-provided input. Preconditions check assumptions before a resource or data-source operation proceeds. Postconditions check guarantees about the resulting resource or data-source value. Choose the mechanism closest to the condition being enforced.

</details>

### 66. Why should module outputs have descriptions?

<details><summary>Answer</summary>

Descriptions tell callers what a value represents and how it should be used. Outputs are part of the module's public interface, so clear descriptions reduce guesswork and improve maintainability.

</details>

### 67. Does marking an output as sensitive remove it from Terraform state?

<details><summary>Answer</summary>

No. The sensitive flag redacts normal CLI display, but the value can still be stored in Terraform state. Secure the backend and restrict state access. Do not treat the flag as encryption or secret removal.

</details>

### 68. Why must a child output sometimes be re-exported by the root module?

<details><summary>Answer</summary>

A parent can reference a child output with `module.<name>.<output>`, but `terraform output` displays outputs declared by the root module. If a root caller needs the child value exposed as a root output, declare a root output that references it.

</details>

### 69. What should a reusable module README contain?

<details><summary>Answer</summary>

It should explain the module purpose, resources created, usage example, required and optional inputs, types and defaults, outputs, provider requirements, permissions, security considerations, assumptions, and validation/test commands. Documentation must stay synchronized with the actual HCL.

</details>

### 70. How would you safely recover from invalid input or module drift?

<details><summary>Answer</summary>

For invalid input, trace the value to its source, compare it with the variable type and validation rules, correct the caller, and re-run validation and plan. For drift, identify the exact module-qualified address, compare configuration, state, and live infrastructure, determine ownership, and review the smallest safe correction. Do not delete state or hide unexplained differences with `ignore_changes`.

</details>


## DAY 8 — Provider Aliases and Multi-Account Patterns

### 71. What is a Terraform provider alias?

<details><summary>Answer</summary>

A provider alias is a named instance of a provider configuration. It lets one Terraform root module configure multiple contexts, such as different AWS accounts or regions. A resource selects an alias using its `provider` meta-argument.

</details>

### 72. Does `alias = "prod"` automatically select the production AWS account?

<details><summary>Answer</summary>

No. The alias is only a Terraform configuration name. The credentials or assumed role configured in that provider determine the actual AWS identity. Verify it with an `aws_caller_identity` data source before applying.

</details>

### 73. How do you create resources in two AWS accounts from one root module?

<details><summary>Answer</summary>

Configure two AWS provider instances, usually with different cross-account role ARNs, and select them explicitly on resources using `provider = aws.dev` or `provider = aws.prod`. Verify the account IDs through STS identity data sources and review the plan before applying.

</details>

### 74. What is the difference between a role trust policy and a role permissions policy?

<details><summary>Answer</summary>

The trust policy defines which principals are allowed to assume the role. The permissions policy defines what the assumed role can do after it is assumed. Cross-account access normally requires both a permitted assume-role call from the source identity and a trust relationship in the target account.

</details>

### 75. How do you verify which AWS account a provider alias targets?

<details><summary>Answer</summary>

Create an `aws_caller_identity` data source with the relevant `provider` argument and inspect its account ID. Do not rely on the alias label, profile name, or expected role ARN alone.

</details>

### 76. How do you pass a provider alias to a child module?

<details><summary>Answer</summary>

The root module passes the configured provider through the module's `providers` map. If the child refers to a named alias such as `aws.target`, it must declare that alias in `configuration_aliases` in its `required_providers` block. The child should not hardcode account credentials.

</details>

### 77. Do provider aliases appear in Terraform resource addresses?

<details><summary>Answer</summary>

No. A resource address remains something like `aws_s3_bucket.prod` or `module.prod_bucket.aws_s3_bucket.this`. Provider configuration association is tracked separately. Use configuration, `terraform providers`, state inspection, and identity data sources to investigate provider selection.

</details>

### 78. What happens if a resource omits its provider argument?

<details><summary>Answer</summary>

It uses the default, unaliased provider configuration when one exists. If all configured provider blocks are aliased and no default is defined, Terraform may use an implied empty default configuration, which can fail due to missing settings. Explicit provider selection is safer for multi-account resources.

</details>

### 79. How would you troubleshoot an STS AssumeRole AccessDenied error?

<details><summary>Answer</summary>

Verify the source identity, its permission to call `sts:AssumeRole` on the exact role ARN, the target role's trust policy, and any external ID, MFA, session-tag, or organization conditions. Then inspect the assumed role's permissions, SCPs, permissions boundaries, and explicit denies. Fix the missing boundary rather than granting broad administrator access.

</details>

### 80. What should you do if Terraform plans changes in the wrong AWS account?

<details><summary>Answer</summary>

Stop before apply. Verify account IDs and regions for each provider alias, inspect resource and module provider mappings, check the backend/workspace, and compare configuration, state, and live resources. Correct the mapping or plan a separately reviewed account migration. Do not delete state or assume changing an alias safely moves existing infrastructure.

</details>


## DAY 9 — Remote State with S3 Backend and State Locking

### 81. Why use a remote backend for Terraform state?

<details><summary>Answer</summary>

A remote backend gives a team and its CI/CD pipelines a shared state location instead of separate local copies. It supports controlled access, collaboration, and centralized state protection. State can contain sensitive values, so remote storage must be secured.

</details>

### 82. What is the purpose of the S3 backend?

<details><summary>Answer</summary>

The S3 backend stores Terraform state as an object in an S3 bucket. The backend configuration identifies the bucket, key, and region. It does not create the bucket; the bucket must exist before backend initialization.

</details>

### 83. How does native S3 state locking work?

<details><summary>Answer</summary>

With `use_lockfile = true`, the S3 backend creates a lock object alongside the state object, using the `.tflock` suffix. Competing Terraform operations cannot acquire the same state lock at the same time. Native S3 lockfiles require Terraform 1.10 or later.

</details>

### 84. Is S3 versioning the same as state locking?

<details><summary>Answer</summary>

No. Versioning preserves prior object versions and helps recover from accidental overwrites or deletions. Locking coordinates concurrent writers. Enable both; neither replaces the other.

</details>

### 85. What IAM permissions are needed for an S3 backend with lockfiles?

<details><summary>Answer</summary>

Typically, `s3:ListBucket` on the bucket with a suitable prefix condition; `s3:GetObject` and `s3:PutObject` on the state object; and `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` on the lock object. Exact permissions depend on backend configuration, workspace use, bucket policy, and encryption design.

</details>

### 86. Why does `s3:ListBucket` use a different ARN from `s3:GetObject`?

<details><summary>Answer</summary>

`s3:ListBucket` is a bucket-level action and uses the bucket ARN, such as `arn:aws:s3:::company-terraform-state`. Object actions such as GetObject and PutObject use object ARNs, such as `arn:aws:s3:::company-terraform-state/path/terraform.tfstate`.

</details>

### 87. What is the purpose of S3 Block Public Access, encryption, and versioning?

<details><summary>Answer</summary>

Block Public Access prevents public ACL or policy exposure. Encryption protects stored state at rest. Versioning preserves previous object versions for recovery. These controls address different risks and should be combined with least-privilege IAM and bucket policies.

</details>

### 88. How do you troubleshoot an S3 backend AccessDenied error?

<details><summary>Answer</summary>

Identify the actual AWS principal with STS, then determine whether the failed operation is ListBucket, GetObject, PutObject, lockfile access, or KMS. Match the action and resource ARN to identity policies, bucket policy, endpoint policy, SCPs, permissions boundaries, and KMS key policy. Fix the missing permission or conflicting condition rather than granting broad S3 access.

</details>

### 89. What is the difference between `terraform init -migrate-state` and `terraform init -reconfigure`?

<details><summary>Answer</summary>

`-migrate-state` asks Terraform to copy existing state to the newly configured backend. `-reconfigure` initializes the working directory with the new backend configuration without migrating the previous state. Choose based on whether state must be moved, and back up state before migration.

</details>

### 90. When is `terraform force-unlock` appropriate?

<details><summary>Answer</summary>

Only when you have verified that the lock belongs to an operation that is no longer running and your team procedure permits unlocking it. Force-unlocking a valid active operation can allow concurrent state modifications and corrupt state. A slow pipeline is not sufficient reason to force-unlock.

</details>


## DAY 10 — State Isolation for Dev, QA and Production

### 91. Why should dev, QA, and production use separate Terraform states?

<details><summary>Answer</summary>

Separate states isolate each environment's resource mapping, plans, locking, and lifecycle. A dev change should not modify production state or create a shared-state blast radius. Separate states also support different deployment schedules and approvals. They should be combined with separate credentials and, preferably, separate AWS accounts for production.

</details>

### 92. Can dev, QA, and prod use the same Terraform configuration?

<details><summary>Answer</summary>

Yes. Reuse the same root module or reusable child modules, but supply environment-specific variables, backend configuration, and credentials. The code can be identical while each environment has a separate state key and potentially a different AWS account.

</details>

### 93. What is the difference between a Terraform workspace and a separate backend key?

<details><summary>Answer</summary>

Workspaces create multiple state instances within one configured backend. Separate backend keys explicitly select different remote state objects. Workspaces can be convenient for similar environments, but they do not automatically provide different credentials or AWS accounts and are not a security boundary. Separate keys and roles are often clearer for production controls.

</details>

### 94. Does using separate S3 state keys guarantee production isolation?

<details><summary>Answer</summary>

No. Different keys isolate state objects, but a role with broad S3 permissions may still access all keys, and AWS credentials may still have permission to modify production resources. Restrict state-prefix access, use separate deployment roles, and preferably place production in a separate AWS account.

</details>

### 95. Why can the same Terraform resource address exist in multiple environment states?

<details><summary>Answer</summary>

A resource address is a logical identity within a particular Terraform state. For example, `aws_s3_bucket.app` can exist in both dev and prod states while mapping to different physical bucket names. The state key determines which environment's mapping Terraform reads.

</details>

### 96. How do you verify that a pipeline is targeting the correct environment?

<details><summary>Answer</summary>

Verify the backend bucket and key, the variable file, the selected workspace if used, and the AWS identity with `aws sts get-caller-identity`. Inspect `terraform state list`, the relevant `terraform state show` output, and the full plan. Do not rely only on a folder or pipeline name.

</details>

### 97. What should you do if a QA pipeline accidentally uses the dev backend?

<details><summary>Answer</summary>

Stop before apply. Identify the actual backend key and AWS principal, then initialize a clean QA working directory against the correct QA key. Use `terraform init -reconfigure` when selecting the correct existing backend without migrating the wrongly selected state. Do not blindly use `-migrate-state`, because it may copy dev state into QA. Inspect state and generate a fresh plan before proceeding.

</details>

### 98. Does changing the backend key change the physical infrastructure?

<details><summary>Answer</summary>

Changing the backend key changes which state object Terraform reads and writes; it does not itself move or rename AWS resources. If the selected state is empty or belongs to another environment, Terraform may propose creating or destroying resources. Review the state mapping and plan before applying.

</details>

### 99. How would you troubleshoot drift in dev without affecting production?

<details><summary>Answer</summary>

Initialize the dev backend in an isolated working directory, verify the AWS account and role, then run a plan and identify the exact resource address. Reconcile the drift according to the agreed ownership model. Run QA and production plans separately to confirm they remain unchanged. Separate state keys and credentials prevent the dev operation from writing to production state.

</details>

### 100. What are the key production guardrails for Terraform state isolation?

<details><summary>Answer</summary>

Use a separate production AWS account and deployment role, restrict S3 state access to the required prefix, protect the state bucket with encryption/versioning/public-access controls/TLS and locking, isolate CI working directories, require protected approvals, verify account identity and backend key, and review the saved plan before apply.

</details>


## DAY 11 — Import, State MV and State RM

### 101. What does terraform import do?

<details><summary>Answer</summary>

It associates an existing remote object with a Terraform resource address in state. It does not create the object and does not automatically generate the complete resource configuration. After import, inspect the state and run a plan to reconcile the HCL with the real object.

</details>

### 102. What are the two arguments to terraform import?

<details><summary>Answer</summary>

The first is the Terraform resource address, such as aws_s3_bucket.app. The second is the provider-specific import identifier for the existing object, such as an S3 bucket name. The required identifier varies by resource type and provider.

</details>

### 103. What is the difference between CLI import and an import block?

<details><summary>Answer</summary>

The CLI terraform import command directly adds the binding to state. An import block declares the target address and remote ID in configuration so the import can be reviewed through terraform plan and applied. Both adopt existing objects; choose the workflow that matches the team's Terraform version and review process.

</details>

### 104. What does terraform state mv do?

<details><summary>Answer</summary>

It changes the Terraform address associated with an object in state without deleting or recreating the real infrastructure. It is useful when renaming a resource label, moving a resource into a module, or changing an instance address. The destination configuration must represent the same object.

</details>

### 105. What is the difference between terraform state mv and a moved block?

<details><summary>Answer</summary>

terraform state mv performs a direct state operation from the CLI. A moved block declares the old and new addresses in configuration, making the migration source-controlled and reviewable in a plan. For normal code refactors, moved blocks are often preferable; state mv remains useful for controlled state surgery.

</details>

### 106. Does terraform state rm delete the actual AWS resource?

<details><summary>Answer</summary>

No. It removes the resource binding from the current Terraform state while leaving the real object in place. If the resource block remains in configuration, a subsequent plan will normally propose creating a new object. Remove or transfer the configuration and ownership deliberately.

</details>

### 107. How do you safely import an existing resource?

<details><summary>Answer</summary>

Verify the provider/account context and remote object ID, ensure the target address is not already tracked, declare matching resource configuration, back up state, and import the object. Then inspect terraform state show and run terraform plan. Reconcile any differences before applying.

</details>

### 108. Why might terraform plan show destroy and create after a resource rename?

<details><summary>Answer</summary>

Terraform sees the old address as no longer configured and the new address as a new resource unless it is told that the addresses represent the same object. Use a correct moved block or controlled terraform state mv, then run a fresh plan and verify that no physical replacement is proposed.

</details>

### 109. What happens if you run terraform state rm but leave the resource block in HCL?

<details><summary>Answer</summary>

The resource remains declared in configuration but is no longer tracked in state. Terraform normally proposes creating a new resource to satisfy that configuration. Do not apply blindly; either re-import the existing object or remove/refactor the old declaration as part of a coordinated ownership transfer.

</details>

### 110. What precautions should you take before state surgery?

<details><summary>Answer</summary>

Back up state securely, stop concurrent operations, respect backend locking, verify the account/backend/workspace, inspect exact source and destination addresses, confirm physical object identity, and run a fresh plan afterward. State backups may contain secrets and must never be committed to Git. Never use state rm as a shortcut for destroying infrastructure.

</details>

### 111. What is Terraform drift?

<details><summary>Answer</summary>

Drift is a difference between the real infrastructure and the desired configuration or recorded state, commonly caused by manual console/CLI changes or another automation system modifying a Terraform-managed resource. Terraform detects drift for attributes it can refresh and manage through the provider.

</details>

### 112. How does Terraform detect drift?

<details><summary>Answer</summary>

During a normal plan, Terraform refreshes managed resource information from provider APIs and compares the refreshed values with configuration. It then proposes actions to reconcile differences. Detection is limited to attributes the provider exposes and Terraform manages; a plan is not a complete audit of every cloud setting.

</details>

### 113. What is the difference between a normal plan and a refresh-only plan?

<details><summary>Answer</summary>

A normal `terraform plan` compares refreshed remote values with HCL and can propose infrastructure changes. `terraform plan -refresh-only` previews changes to Terraform state so it can reflect remote reality, without proposing changes to the remote infrastructure. `terraform apply -refresh-only` records those refreshed values in state but does not update HCL.

</details>

### 114. If someone changes an AWS tag manually, what should you do?

<details><summary>Answer</summary>

Identify the exact resource address and physical object, verify account/backend/workspace, and determine who owns the tag. If Terraform owns it, review a plan that restores the declared value and apply through the approved workflow. If the change is approved, update HCL or deliberately accept the remote value with a refresh-only workflow. Do not blindly apply.

</details>

### 115. Does terraform apply -refresh-only fix infrastructure drift?

<details><summary>Answer</summary>

No. It updates Terraform's state snapshot to reflect remote values; it does not change the remote resource to match HCL. If configuration still differs, a later normal plan can propose changing the infrastructure again. Update HCL too when the remote change is approved as the new desired state.

</details>

### 116. When is lifecycle ignore_changes appropriate?

<details><summary>Answer</summary>

When another system intentionally owns specific attributes and Terraform should not reconcile those attributes. Scope the rule narrowly, document ownership, and preserve monitoring and security controls. Broadly ignoring attributes merely to hide drift can conceal unauthorized or risky changes.

</details>

### 117. What would you do if a production drift plan shows resource replacement?

<details><summary>Answer</summary>

Stop before apply. Verify AWS identity, region, backend key, workspace, provider alias, exact address, physical ID, and the attribute forcing replacement. Investigate the change history and data/downtime impact, determine the source of truth, and obtain the required owner approval. Generate a fresh reviewed plan before proceeding.

</details>

### 118. Why can terraform plan show no changes even when the cloud account has unmanaged changes?

<details><summary>Answer</summary>

Terraform only evaluates resources and attributes represented by the current configuration/state and exposed through provider read operations. Unmanaged resources, ignored attributes, unsupported fields, eventual consistency, or the wrong account/workspace/backend can prevent a particular change from appearing. A Terraform plan is not a full cloud security or configuration audit.

</details>

### 119. Should you use terraform state rm to resolve drift?

<details><summary>Answer</summary>

No. `terraform state rm` removes Terraform's binding to an object; it does not reconcile the remote resource. If the resource block remains, Terraform may propose creating another object. Use state removal only for a deliberate ownership handoff or state-management operation.

</details>

### 120. Explain your drift reconciliation workflow.

<details><summary>Answer</summary>

Detect drift with a normal plan, identify the exact Terraform address and physical object, verify account/backend/workspace, inspect the changed attributes and audit history, and determine which system owns the change. Review the proposed action and its impact. Reconcile HCL or the remote resource according to that ownership decision, apply through the approved workflow, and verify with a fresh plan and state inspection.

</details>


## DAY 13 — Sensitive Values and Secrets Handling

### 121. What does sensitive = true do in Terraform?

<details><summary>Answer</summary>

It marks a value as sensitive so Terraform redacts it from normal CLI output and supported UI displays. It does not guarantee that the value is omitted from state or saved plan files. Treat persisted state and plans as sensitive artifacts.

</details>

### 122. What is the difference between sensitive and ephemeral values?

<details><summary>Answer</summary>

Sensitive controls display: Terraform hides the value in normal output. Ephemeral controls persistence: Terraform omits the value from state and plan files. They can be combined so a runtime-only secret is both redacted and not persisted. Ephemeral values are restricted to supported contexts.

</details>

### 123. Which Terraform versions introduced ephemeral values and write-only resource arguments?

<details><summary>Answer</summary>

Terraform 1.10 introduced ephemeral input variables, child-module ephemeral outputs, and ephemeral resources. Terraform 1.11 introduced support for write-only managed-resource arguments. Check the provider's resource documentation because write-only support is implemented per resource.

</details>

### 124. Why can a sensitive variable still appear in Terraform state?

<details><summary>Answer</summary>

The sensitive flag is primarily a redaction feature. If the value is passed to a normal resource argument or returned by a normal data source, the provider may persist it in state. Use a supported ephemeral input and write-only argument when the value must not be persisted; otherwise secure state and plans as secrets.

</details>

### 125. What is a write-only argument?

<details><summary>Answer</summary>

A provider-defined managed-resource argument that Terraform can send during an operation but does not store in plan or state. It is useful for passwords and tokens. Because Terraform cannot read or compare the value later, providers commonly pair it with a normal version argument that signals when the secret should be updated.

</details>

### 126. Why does an SSM SecureString not automatically protect Terraform state?

<details><summary>Answer</summary>

SecureString encrypts the parameter at rest in AWS Systems Manager Parameter Store. If Terraform writes the value through the ordinary value argument, the provider can still place the plaintext value in Terraform state. Use the supported value_wo write-only argument with Terraform 1.11+ and a compatible AWS provider to avoid persisting the value.

</details>

### 127. How do you rotate a secret managed through a write-only argument?

<details><summary>Answer</summary>

Update the secret in the approved secret source, inject the matching new value into the Terraform run, and increment the associated version argument such as value_wo_version. Review the plan and apply through the approved pipeline. Terraform cannot compare old and new write-only values, so the version is the change signal.

</details>

### 128. Is it safe to read a secret from Secrets Manager using a normal Terraform data source?

<details><summary>Answer</summary>

Not automatically. A normal data source can return the decrypted value, which may then be persisted in state even if Terraform redacts it in CLI output. Prefer an ephemeral provider resource where supported, or let the application workload retrieve the secret directly using its IAM role. Verify the exact provider and Terraform version.

</details>

### 129. What should you do if a secret is accidentally committed to Git?

<details><summary>Answer</summary>

Treat it as compromised and rotate or revoke it immediately. Then identify all exposure locations including Git history, clones, CI logs, artifacts, state, plans, caches, and backups. Restrict access, remove it from history using the approved procedure, secure affected artifacts, and review audit logs. Deleting the current file is not enough.

</details>

### 130. Explain your production Terraform secrets-handling strategy.

<details><summary>Answer</summary>

Keep secrets out of HCL and source control. Prefer workload identity or short-lived credentials for provider authentication. Inject runtime secrets from an approved secret store. Use sensitive for output redaction, ephemeral values and supported write-only arguments to prevent persistence, and let workloads retrieve application secrets directly when feasible. Encrypt and restrict state and plan access, protect CI logs and artifacts, implement rotation, and verify provider-specific behavior before production use.

</details>


## DAY 14 — Terraform fmt, validate, plan and CI Quality Gates

### 131. What is the difference between terraform fmt, validate and plan?

<details><summary>Answer</summary>

fmt standardizes HCL formatting. validate checks configuration syntax, references, types and internal consistency after initialization. plan evaluates the proposed infrastructure actions using configuration, state and provider reads. Passing one does not imply the others will pass or that a change is safe to apply.

</details>

### 132. Why should CI use terraform fmt -check instead of terraform fmt?

<details><summary>Answer</summary>

fmt -check verifies formatting and exits unsuccessfully when files need changes, without modifying the pull-request workspace. A developer can run terraform fmt locally, inspect the diff and commit the result. This makes formatting failures visible and reproducible.

</details>

### 133. What does terraform init -backend=false do in a validation pipeline?

<details><summary>Answer</summary>

It initializes the working directory and installs required providers/modules without initializing the configured backend. It is useful for syntax/validation jobs that should not access remote state. It does not prove backend credentials, state access or remote locking work.

</details>

### 134. Does terraform validate prove that infrastructure can be deployed?

<details><summary>Answer</summary>

No. validate checks configuration consistency, not whether AWS credentials are valid, quotas are available, APIs are reachable, permissions allow the operation, or the resulting changes are operationally safe. A plan and environment-specific checks are still required.

</details>

### 135. What should a basic Terraform pull-request quality gate contain?

<details><summary>Answer</summary>

A common baseline is a pinned Terraform CLI version, checkout, terraform fmt -check -recursive, terraform init -backend=false and terraform validate. A separate trusted plan job can inspect real infrastructure changes when it has safe backend and cloud access. Apply should be protected by approvals and environment controls.

</details>

### 136. Why should an untrusted fork pull request not receive cloud credentials?

<details><summary>Answer</summary>

Pull-request code can execute commands in CI. Exposing cloud credentials or privileged secrets to untrusted code can allow an attacker to access infrastructure or state. Use least privilege, short-lived OIDC credentials, trusted-event restrictions and protected environments; do not issue privileged credentials to untrusted fork workflows.

</details>

### 137. Why are Terraform plan files sensitive?

<details><summary>Answer</summary>

Saved plans can contain configuration and planned values, including sensitive data. Restrict artifact access and retention, avoid public uploads and never commit plan files to source control. Treat them with controls similar to Terraform state.

</details>

### 138. What does a replacement marker in a plan mean even if validate passes?

<details><summary>Answer</summary>

Validation only checks configuration consistency. A replacement marker means the provider requires the object to be destroyed and recreated for the proposed change. Stop and inspect the exact address and replacement-causing attribute, operational impact and environment before approval.

</details>

### 139. Why should production apply be separate from pull-request validation?

<details><summary>Answer</summary>

Validation and plan are review gates, not deployment authorization. Production apply needs trusted credentials, protected state access, an approved plan, change approval and auditability. Separating it prevents an ordinary pull request from deploying infrastructure automatically.

</details>

### 140. Explain your Terraform CI quality-gate strategy.

<details><summary>Answer</summary>

I pin the Terraform CLI and provider versions, run fmt check, initialize without backend access for static validation, and run validate on pull requests. A separate trusted plan job uses short-lived identity and least-privilege state/cloud permissions. Reviewers inspect resource addresses, replacement actions and sensitive plan artifacts. Production apply is a separate protected stage with approvals, followed by verification.

</details>


## DAY 15 — Terraform Troubleshooting: Dependency Graph and Failed Plans

### 141. What is Terraform's dependency graph?

<details><summary>Answer</summary>

It is a directed graph Terraform builds to determine operation ordering and safe parallelism. A resource reference normally creates an implicit dependency. Terraform can process independent nodes in parallel while waiting for dependencies to complete.

</details>

### 142. How does Terraform infer an implicit dependency?

<details><summary>Answer</summary>

When one resource expression references an attribute from another resource, Terraform knows the referenced resource must be available first. For example, bucket versioning referencing aws_s3_bucket.app.id creates a dependency on aws_s3_bucket.app without an explicit depends_on.

</details>

### 143. When should you use depends_on?

<details><summary>Answer</summary>

Use depends_on only when a real dependency exists but Terraform cannot infer it from an expression reference, such as an ordering requirement involving a side effect. Keep it narrowly scoped. Unnecessary dependencies can reduce parallelism and make plans more conservative.

</details>

### 144. How do you inspect Terraform's dependency graph?

<details><summary>Answer</summary>

Run terraform graph to emit DOT graph data. Save it to a file and render with Graphviz, for example terraform graph > graph.dot followed by dot -Tpng graph.dot -o graph.png. The graph includes internal nodes as well as resources, so focus on relevant dependency edges.

</details>

### 145. What causes a Terraform dependency cycle?

<details><summary>Answer</summary>

A cycle occurs when resources depend on each other directly or through a chain of other resources, so Terraform cannot determine a valid operation order. Inspect the cycle diagnostic and graph, identify the circular reference or depends_on edge, then redesign the dependency or split the operation into valid stages.

</details>

### 146. What causes an invalid index error during planning?

<details><summary>Answer</summary>

An expression refers to an index that does not exist in a list or resource instance collection, often after count, conditional creation or input changes. Inspect collection length and instance addresses, then use a valid index or stable for_each keys. Do not guess an index just to silence the error.

</details>

### 147. Are unknown values in a Terraform plan always errors?

<details><summary>Answer</summary>

No. Computed attributes often remain unknown until an upstream resource is created. This is normal when downstream resources reference those attributes. However, values used to determine instance identity, such as for_each keys, generally must be known during planning; restructure the configuration if they are not.

</details>

### 148. What is the difference between a failed plan and a failed apply?

<details><summary>Answer</summary>

A failed plan means Terraform could not produce a valid proposed action set; investigate configuration, dependencies, provider reads, credentials or APIs before applying. A failed apply can occur after some operations succeeded, so inspect both Terraform state and real infrastructure before retrying.

</details>

### 149. What should you do when apply fails after creating some resources?

<details><summary>Answer</summary>

Do not assume rollback. Identify the failed address and provider error, verify account/backend/workspace, inspect state and check the real cloud objects. Fix the root cause, generate a fresh plan and review remaining actions before retrying. Avoid state removal or repeated blind applies.

</details>

### 150. Explain your workflow for troubleshooting a failed Terraform plan.

<details><summary>Answer</summary>

I identify the failing phase and exact diagnostic/address, verify CLI/provider versions, account, region, backend and workspace, then inspect references, dependencies, inputs and provider permissions. If needed, I inspect terraform graph and state. I fix the root cause, run fmt and validate, generate a fresh plan and verify that only intended actions remain before any apply.

</details>


## DAY 16 — Terraform Testing Concepts and Policy Checks

### 151. What is the difference between terraform validate and terraform test?

<details><summary>Answer</summary>

validate checks whether the initialized configuration is structurally valid, including syntax, references and types. terraform test executes test cases written in .tftest.hcl files and checks assertions or expected failures. Tests verify intended module behavior, not just configuration structure.

</details>

### 152. What is a .tftest.hcl file?

<details><summary>Answer</summary>

It is a Terraform native test file containing run blocks, variables, command modes, assertions and expected failures. Terraform discovers these files as part of terraform test. The native testing framework became available in Terraform 1.6.

</details>

### 153. What is the difference between plan-mode and apply-mode Terraform tests?

<details><summary>Answer</summary>

A plan-mode test evaluates proposed values and assertions without creating managed infrastructure. An apply-mode test performs real resource operations and can incur cost or leave resources behind if cleanup fails. Use plan mode for safe module logic checks and isolated disposable environments for apply-mode integration tests.

</details>

### 154. What are mock providers in Terraform testing?

<details><summary>Answer</summary>

Mock providers simulate provider behavior and can generate computed values for tests, allowing module logic to be tested without real cloud operations. Provider mocking was introduced in Terraform 1.7. Mock success does not prove real IAM permissions, API behavior, quotas or service availability.

</details>

### 155. What is policy as code in Terraform?

<details><summary>Answer</summary>

Policy as code expresses organizational rules as machine-evaluable policies. It can reject plans that violate controls such as public S3 access, missing tags, unapproved regions or prohibited destructive actions. Sentinel and OPA are common policy approaches.

</details>

### 156. What is the difference between Sentinel and OPA?

<details><summary>Answer</summary>

Sentinel is HashiCorp's policy-as-code framework used with supported HashiCorp platforms. OPA is a general-purpose policy engine that uses Rego and can evaluate many kinds of structured input, including Terraform plan JSON. The choice depends on the platform, integration and enforcement requirements.

</details>

### 157. Why convert a Terraform plan to JSON for policy evaluation?

<details><summary>Answer</summary>

terraform show -json exposes structured planned resource changes, addresses, actions and values so a policy engine can evaluate the proposed infrastructure rather than relying only on source text. The JSON may contain sensitive values, so it must be protected like state and saved plan artifacts.

</details>

### 158. Can a Terraform test pass while a real AWS apply fails?

<details><summary>Answer</summary>

Yes. A plan-mode or mocked test may verify module logic without creating real resources. A real apply can still fail due to IAM permissions, quotas, invalid account configuration, network issues or provider/API errors. Use controlled integration tests and verify the actual deployment.

</details>

### 159. What should you do when a policy check rejects a valid Terraform plan?

<details><summary>Answer</summary>

Do not bypass it immediately. Identify the policy rule and resource address, inspect the plan and determine whether the violation is real or a false positive. Correct the configuration or submit a reviewed policy change with tests and approval. Re-run the full pipeline and document any exception with an owner and expiry.

</details>

### 160. Describe a robust Terraform testing and policy pipeline.

<details><summary>Answer</summary>

I run formatting, initialization, validation and native module tests first. A trusted plan job then generates a plan using least-privilege short-lived credentials. Policy-as-code and security checks evaluate the proposed changes, with plan artifacts protected. Review and approval gate the apply stage. After deployment, I verify real cloud configuration and inspect state. Mock tests are not treated as substitutes for integration tests.

</details>


## DAY 17 — Build a Reusable AWS VPC Terraform Module

### 161. What is the purpose of a reusable VPC Terraform module?

<details><summary>Answer</summary>

A reusable VPC module packages network resources and exposes a stable interface through typed inputs and outputs. Different root configurations can reuse it with different CIDRs, subnet layouts, tags, and routing choices instead of duplicating resource definitions. It should avoid hardcoding environment-specific values and should document its networking and availability trade-offs.

</details>

### 162. What is the difference between a root module and a child module?

<details><summary>Answer</summary>

The root module is the working directory where Terraform is initialized and where the overall deployment is composed. A child module is a reusable configuration called by the root or another module. The root supplies values and provider configuration; the child defines the resources and outputs its consumers need.

</details>

### 163. Why use for_each for public and private subnet resources?

<details><summary>Answer</summary>

for_each creates instances using stable map keys, producing addresses such as module.network.aws_subnet.private["private_a"]. Adding or removing another key does not shift the identity of existing instances as positional count indexes can. This makes subnet changes easier to review and reduces accidental address churn.

</details>

### 164. What makes an AWS subnet public?

<details><summary>Answer</summary>

A subnet is public when its associated route table has a route to an Internet Gateway. For an EC2 instance to communicate directly over the internet, it also needs a public IPv4 address or another valid public addressing mechanism, plus suitable security-group and network ACL rules. The subnet name alone does not make it public.

</details>

### 165. What is the difference between an Internet Gateway and a NAT Gateway?

<details><summary>Answer</summary>

An Internet Gateway provides internet connectivity for resources with suitable public addressing and routing. A NAT Gateway allows resources in private subnets to initiate outbound IPv4 connections while preventing unsolicited inbound internet connections to those private resources. A NAT Gateway is placed in a public subnet and requires a route to an Internet Gateway for internet egress.

</details>

### 166. Why might one NAT Gateway be unsuitable for production across multiple AZs?

<details><summary>Answer</summary>

A single NAT Gateway creates an AZ dependency and can cause cross-AZ traffic and data-transfer charges when private subnets in other AZs use it. It is also a single point of failure for that egress path. A common production design deploys one NAT Gateway per AZ and routes each AZ's private subnets through its local NAT Gateway, while balancing resilience against cost.

</details>

### 167. How do you prevent a VPC module from creating a NAT Gateway without a public subnet?

<details><summary>Answer</summary>

Validate the module inputs and add a resource precondition requiring at least one public subnet whenever NAT is enabled. The plan should fail with a clear message before creating the NAT Gateway. The caller must add a valid public subnet or disable NAT if private-subnet internet egress is not required.

</details>

### 168. How do you inspect resources created by a child module in Terraform state?

<details><summary>Answer</summary>

Use terraform state list to see module-qualified addresses, then terraform state show with the exact address. For example, module.network.aws_subnet.public["public_a"]. The Terraform address identifies the state binding, while the subnet ID is the physical AWS identifier. Use the module outputs for normal downstream consumption rather than coupling callers to internal resource addresses.

</details>

### 169. What happens if you rename a module block that already manages AWS infrastructure?

<details><summary>Answer</summary>

The module-qualified state addresses change, so Terraform may interpret the old resources as no longer managed at the old address and propose creating resources at the new address. Use a reviewed moved block or terraform state mv migration, then inspect a fresh plan to confirm Terraform recognizes the existing infrastructure instead of replacing it.

</details>

### 170. How would you troubleshoot a Terraform-managed subnet tag changed manually in AWS?

<details><summary>Answer</summary>

Run terraform plan and identify the exact module-qualified subnet address and changed tag. Confirm which system owns that tag. If Terraform owns it, restore the desired value in configuration and review the plan. If an external controller owns it, define a narrow ownership boundary. Do not remove the state binding to fix normal drift; verify the final plan, state, and live AWS tag.

</details>


## DAY 18 — Build EC2/IAM Modules with Clean Interfaces

### 171. Why separate EC2 and IAM into different Terraform modules?

<details><summary>Answer</summary>

They have different responsibilities and lifecycles. The IAM module owns the role, trust policy, permissions, and instance profile. The EC2 module owns compute settings and consumes the profile name as an input. This separation avoids duplicated roles, broad hidden permissions, and tight coupling between compute and identity.

</details>

### 172. What is the difference between an IAM role and an instance profile?

<details><summary>Answer</summary>

An IAM role defines trusted principals and the permissions available after assumption. An EC2 instance profile is the container through which a role is associated with an EC2 instance. EC2 uses the profile to make temporary role credentials available through instance metadata. A role and an instance profile are separate IAM objects.

</details>

### 173. What should the EC2 trust policy contain for an EC2 instance role?

<details><summary>Answer</summary>

The trust policy should allow the EC2 service principal, ec2.amazonaws.com, to perform sts:AssumeRole. This is separate from the role's permissions policy. The trust policy answers who can assume the role; attached permissions answer what the assumed role can do.

</details>

### 174. Why should an EC2 module accept an instance-profile name instead of creating its own IAM role?

<details><summary>Answer</summary>

The profile is an explicit dependency supplied by the caller. This lets IAM lifecycle and permissions be managed independently, supports reuse across compute configurations, and prevents the EC2 module from silently granting permissions. The root module composes the IAM and EC2 modules by passing the IAM module's output into the EC2 module.

</details>

### 175. How does an EC2 instance obtain temporary AWS credentials?

<details><summary>Answer</summary>

The EC2 instance is associated with an instance profile containing an IAM role. The instance metadata service provides temporary credentials for that role, and applications or agents use them through supported SDK credential providers. Require IMDSv2 and never hardcode or log the credential values.

</details>

### 176. What does IMDSv2 protect against, and how do you require it in Terraform?

<details><summary>Answer</summary>

IMDSv2 requires a session token to access instance metadata, helping mitigate certain server-side request forgery and metadata exposure risks. In aws_instance, configure metadata_options with http_endpoint = "enabled" and http_tokens = "required". Review hop-limit requirements for containerized workloads before changing the default design.

</details>

### 177. Why can an EC2 instance be running but not appear in Systems Manager?

<details><summary>Answer</summary>

EC2 running state only confirms that the instance launched. SSM registration additionally requires a healthy SSM Agent, an attached instance profile with the required permissions, working instance metadata credentials, DNS resolution, and outbound connectivity to the regional SSM services. Diagnose each layer instead of assuming the launch was incomplete.

</details>

### 178. How would you troubleshoot an EC2 instance that is offline in Systems Manager?

<details><summary>Answer</summary>

First confirm EC2 state, status checks, subnet, and attached instance profile. Verify the profile contains the intended role, the trust policy allows EC2, and the required SSM permissions are attached. Then inspect IMDSv2 IAM metadata, the SSM Agent service and logs, DNS, routes, NAT or VPC endpoints, endpoint security groups, and outbound TCP 443. Fix the failing layer and verify the instance appears online.

</details>

### 179. Does Session Manager require inbound SSH access to the EC2 instance?

<details><summary>Answer</summary>

No. The SSM Agent initiates outbound connections to the Systems Manager services over HTTPS. A private instance can use NAT or the required VPC interface endpoints. Security groups should allow the necessary egress, while no public inbound SSH rule is required for Session Manager.

</details>

### 180. What is the difference between an EC2 boot failure and an SSM connectivity failure?

<details><summary>Answer</summary>

A boot failure occurs during instance initialization, storage attachment, operating-system startup, or cloud-init. An SSM connectivity failure can occur after the operating system has booted because the agent, instance profile, permissions, DNS, routes, or service endpoints are unavailable. Use EC2 status checks, console/system logs, cloud-init logs, SSM Agent logs, and IAM/network inspection to isolate the layer.

</details>
