# Terraform

Terraform is the infrastructure-as-code track for this repository.

## Progress

- [x] DAY 1 — Professional Workflow: Repo Structure, Providers and Versions
- [x] DAY 2 — Variables, Locals, Outputs and Type Constraints
- [x] DAY 3 — Resource Lifecycle, Dependencies and Replace Behavior
- [x] DAY 4 — `count` vs `for_each` and Stable Resource Addressing
- [x] DAY 5 — Dynamic Blocks, Expressions and Conditional Infrastructure

## Topics

| 6 | Reusable Terraform module design | [DAY 6](DAY%206%20-%20Reusable%20Terraform%20Module%20Design.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |

| Day | Topic | Notes | Labs | Interview |
|---|---|---|---|---|
| 1 | Professional workflow, repo structure, providers and versions | [DAY 1](DAY%201%20-%20Professional%20Workflow%20Repo%20Structure%20Providers%20and%20Versions.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 2 | Variables, locals, outputs and type constraints | [DAY 2](DAY%202%20-%20Variables%20Locals%20Outputs%20and%20Type%20Constraints.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 3 | Resource lifecycle, dependencies and replace behavior | [DAY 3](DAY%203%20-%20Resource%20Lifecycle%20Dependencies%20and%20Replace%20Behavior.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 4 | `count` vs `for_each` and stable resource addressing | [DAY 4](DAY%204%20-%20Count%20vs%20For%20Each%20and%20Stable%20Resource%20Addressing.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 5 | Dynamic blocks, expressions and conditional infrastructure | [DAY 5](DAY%205%20-%20Dynamic%20Blocks%20Expressions%20and%20Conditional%20Infrastructure.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |

## Centralized files

- [LABS.md](LABS.md) — all Terraform practical labs
- [INTERVIEW QUESTIONS.md](INTERVIEW%20QUESTIONS.md) — all Terraform interview questions

## Day 3 focus

- Understand Terraform lifecycle meta-arguments: `create_before_destroy`, `prevent_destroy`, and `ignore_changes`.
- Distinguish in-place updates from replacement behavior such as `-/+`.
- Understand implicit dependencies and when `depends_on` is justified.
- Inspect dependency graphs with `terraform graph`.
- Inspect exact resource addresses and state with `terraform state list` and `terraform state show`.
- Save and inspect reviewed plans with `terraform plan -out` and `terraform show`.
- Recover safely from accidental replacement and drift without deleting Terraform state.


## Day 4 focus

- Compare indexed `count` addresses with keyed `for_each` addresses.
- Understand why removing a middle list item can shift `count` instance identity.
- Use `each.key` and `each.value` with sets and maps.
- Inspect exact addresses with `terraform state list` and `terraform state show`.
- Migrate addresses safely with `moved` blocks or a carefully reviewed `terraform state mv`.
- Verify that address changes do not cause unintended resource replacement.


## Day 5 focus

- Use conditional expressions and `for` expressions to calculate and transform values.
- Use functions such as `lookup`, `merge`, `try`, and `coalesce` appropriately.
- Create optional resources with conditional `count`.
- Generate repeatable nested configuration with dynamic blocks.
- Inspect AWS security-group ingress rules and optional CloudWatch alarms in the plan.
- Troubleshoot malformed input objects and validate generated network rules before applying.


## Day 6 focus

- Separate root-module responsibilities from child-module responsibilities.
- Design typed inputs, validation rules, and useful outputs.
- Build and call a local reusable AWS S3 module.
- Understand provider inheritance and module source/version choices.
- Inspect module-qualified resource addresses in plans and state.
- Reuse one module from multiple module blocks without confusing their identities.
- Recover safely from invalid inputs, module address changes, and out-of-band drift.
