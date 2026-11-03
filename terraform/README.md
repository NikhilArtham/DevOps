# Terraform

Terraform is the infrastructure-as-code track for this repository.

## Progress

- [x] DAY 1 — Professional Workflow: Repo Structure, Providers and Versions
- [x] DAY 2 — Variables, Locals, Outputs and Type Constraints
- [x] DAY 3 — Resource Lifecycle, Dependencies and Replace Behavior

## Topics

| Day | Topic | Notes | Labs | Interview |
|---|---|---|---|---|
| 1 | Professional workflow, repo structure, providers and versions | [DAY 1](DAY%201%20-%20Professional%20Workflow%20Repo%20Structure%20Providers%20and%20Versions.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 2 | Variables, locals, outputs and type constraints | [DAY 2](DAY%202%20-%20Variables%20Locals%20Outputs%20and%20Type%20Constraints.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |
| 3 | Resource lifecycle, dependencies and replace behavior | [DAY 3](DAY%203%20-%20Resource%20Lifecycle%20Dependencies%20and%20Replace%20Behavior.md) | [LABS](LABS.md) | [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md) |

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
