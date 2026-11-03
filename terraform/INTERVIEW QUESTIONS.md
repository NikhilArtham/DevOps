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
