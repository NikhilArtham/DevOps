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
