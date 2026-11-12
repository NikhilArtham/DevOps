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
