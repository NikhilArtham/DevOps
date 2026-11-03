# DAY 3 — Resource Lifecycle, Dependencies and Replace Behavior

## 1. Why resource lifecycle matters

Terraform normally decides whether a resource can be updated in place or must be replaced. A replacement means the existing object is destroyed and a new object is created, although the exact ordering depends on the resource and lifecycle settings.

In production, replacement behavior matters because a plan that says `-/+` is materially different from an in-place `~` update. Before approving a replacement, identify the exact resource address, why replacement is required, and whether the downtime or data-loss risk is acceptable.

## 2. Resource lifecycle meta-argument

Terraform resources can use the `lifecycle` meta-argument to change how Terraform handles changes.

### `create_before_destroy`

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}
```

This asks Terraform to create the replacement before destroying the old resource when the provider and resource constraints allow it.

Typical use case: replacing compute capacity where the old instance can remain available while the new one is created.

Important limitation: this is not a universal zero-downtime switch. Names, unique identifiers, quotas, dependencies, and provider behavior can prevent parallel existence.

### `prevent_destroy`

```hcl
resource "aws_db_instance" "production" {
  # configuration...

  lifecycle {
    prevent_destroy = true
  }
}
```

Terraform will fail the plan/apply if the proposed operation would destroy the resource. This is useful as a guardrail for critical infrastructure, but it can also block legitimate changes that require replacement.

### `ignore_changes`

```hcl
resource "aws_instance" "app" {
  # configuration...

  lifecycle {
    ignore_changes = [tags]
  }
}
```

Terraform ignores changes to the specified attributes when calculating differences. Use this carefully: it can intentionally allow another system to manage an attribute, but it can also hide configuration drift.

Do not use `ignore_changes` simply to make an unwanted plan disappear.

## 3. Replace behavior

Terraform can show different plan actions:

- `+` create
- `~` update in place
- `-` destroy
- `-/+` destroy and recreate / replacement
- `+/-` create replacement first, then destroy the old instance when the lifecycle behavior permits it

A replacement normally occurs when an attribute is immutable or the provider marks the changed attribute as requiring replacement.

Example:

```text
# aws_instance.app must be replaced
-/+ resource "aws_instance" "app" {
      ~ ami = "ami-old" -> "ami-new" # forces replacement
}
```

The key interview point is: **Terraform does not replace a resource merely because a value changed; replacement happens because the resource/provider cannot safely update that attribute in place.**

## 4. Dependencies

Terraform builds a dependency graph to determine ordering.

### Implicit dependency

A reference automatically creates a dependency.

```hcl
resource "aws_security_group" "app" {
  name = "app-sg"
}

resource "aws_instance" "app" {
  security_groups = [aws_security_group.app.id]
}
```

Because the instance references `aws_security_group.app.id`, Terraform knows the instance depends on the security group.

### Explicit dependency

Use `depends_on` when Terraform cannot infer a dependency from an expression.

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  depends_on = [aws_iam_role_policy.app]
}
```

Use `depends_on` sparingly. Prefer real references when possible because references communicate the actual relationship and give Terraform a more precise dependency graph.

## 5. `terraform graph`

Use the graph command to inspect the dependency graph:

```bash
terraform graph
```

The output is DOT-format graph data. It can be useful when troubleshooting why Terraform is ordering resources unexpectedly.

The exact resource addresses remain important. For example:

```text
aws_security_group.app
aws_instance.app
aws_iam_role_policy.app
```

## 6. Inspecting plan and state

Always inspect the plan before applying a replacement:

```bash
terraform plan
```

Useful state commands:

```bash
terraform state list
terraform state show aws_instance.app
```

`terraform state list` shows the resource addresses currently tracked in state. `terraform state show` shows the stored attributes for a specific address.

To inspect a specific plan without applying it:

```bash
terraform plan -out=tfplan
terraform show tfplan
```

The `-out=tfplan` option saves the generated plan so the reviewed plan can be used later with `terraform apply tfplan`.

## 7. Realistic production scenario — accidental replacement

Imagine a production EC2 resource:

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.medium"
}
```

A developer changes the AMI ID and the plan shows:

```text
-/+ aws_instance.app
```

Before applying, investigate:

1. Confirm the exact address: `aws_instance.app`.
2. Identify the changed attribute causing replacement.
3. Confirm whether the AMI change is intentional.
4. Check whether the workload has persistent state outside the instance.
5. Check load balancer/target-group behavior and capacity requirements.
6. Decide whether `create_before_destroy` is safe for this resource.
7. Run the plan again after the intended configuration change.
8. Apply only after the replacement impact is understood.

If the AMI change was accidental, revert the configuration and re-run `terraform plan`. Do not delete state just because the plan contains a replacement.

## 8. Realistic failure scenario — dependency is missing

Suppose an application needs an IAM policy to exist before an application resource is created, but the configuration contains no expression connecting the two resources.

Terraform may not know about the operational dependency.

Recovery:

1. Confirm the exact resources involved.
2. Check whether a direct reference can express the relationship.
3. If no reference is appropriate, add a narrowly scoped `depends_on`.
4. Run `terraform plan`.
5. Inspect `terraform graph` if ordering is still unclear.
6. Apply only after confirming the graph and plan match the intended order.

## 9. Drift and lifecycle recovery

A common mistake is to add `ignore_changes` after someone changes a production resource manually and Terraform detects drift.

That treats the symptom rather than the ownership problem.

Instead:

- Decide which system should own the attribute.
- If Terraform owns it, restore the configuration and let Terraform reconcile it.
- If another system intentionally owns it, document the ownership boundary and use `ignore_changes` only where appropriate.
- Re-run `terraform plan` and verify the remaining changes are expected.

## 10. AWS and Kubernetes relevance

The same Terraform lifecycle concepts apply to Kubernetes resources managed through the Kubernetes provider.

For example, changing an immutable Kubernetes field may cause replacement or a provider-specific recreation behavior. Always inspect the generated plan rather than assuming every Kubernetes change is an in-place update.

AWS examples commonly involve EC2, RDS, networking, security groups, launch templates, and load-balancer-related resources. Replacement of stateful resources such as databases deserves substantially more scrutiny than replacement of disposable compute.

## 11. Common mistakes

- Treating every `-/+` replacement as harmless.
- Assuming `create_before_destroy` guarantees zero downtime.
- Adding `depends_on` everywhere instead of using resource references.
- Using `ignore_changes` to hide drift without defining ownership.
- Ignoring quotas or uniqueness constraints when creating replacements.
- Deleting Terraform state to resolve an unexpected replacement.
- Applying without identifying the exact resource address and replacement reason.

## 12. Interview-ready mental model

**Configuration change → dependency graph → provider/resource rules → plan → update or replacement → state reconciliation.**

For an unexpected replacement, the first question is not "How do I stop Terraform?" It is: **"Which exact attribute changed, why does the provider require replacement, and is that replacement safe?"**
