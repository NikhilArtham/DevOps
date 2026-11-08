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
