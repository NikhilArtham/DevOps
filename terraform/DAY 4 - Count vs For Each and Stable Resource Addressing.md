# DAY 4 — `count` vs `for_each` and Stable Resource Addressing

## 1. Why this matters

Terraform needs a stable address for every managed resource instance. The address is how Terraform connects configuration to the object recorded in state.

When you use `count`, Terraform identifies instances by numeric position. When you use `for_each`, Terraform identifies instances by a map key or set member.

This difference matters when a collection changes. Removing an item from the middle of a list can shift numeric indexes and cause Terraform to update or replace the wrong instances. Meaningful `for_each` keys avoid that positional identity problem.

## 2. `count`: create indexed instances

Use `count` when instances are nearly identical and their identity is naturally positional, or when you need a simple on/off switch.

```hcl
variable "instance_count" {
  type    = number
  default = 2
}

resource "aws_instance" "app" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "app-${count.index}"
  }
}
```

With a count of two, the resource addresses are:

```text
aws_instance.app[0]
aws_instance.app[1]
```

Useful expression: `count.index` gives the zero-based index of the current instance.

A common conditional pattern is `count = var.enabled ? 1 : 0`. The resource address exists only when enabled; when disabled, Terraform plans to destroy the instance.

## 3. The positional identity problem with `count`

Suppose the instances are created from this list:

```hcl
variable "names" {
  type    = list(string)
  default = ["api", "worker", "scheduler"]
}

resource "aws_instance" "app" {
  count         = length(var.names)
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = var.names[count.index]
  }
}
```

Addresses initially map as follows:

| Address | Name |
|---|---|
| `aws_instance.app[0]` | api |
| `aws_instance.app[1]` | worker |
| `aws_instance.app[2]` | scheduler |

Now remove `worker` from the middle. The new list is `["api", "scheduler"]`. Terraform still sees indexes 0 and 1. Index 1 now represents scheduler rather than worker, and index 2 disappears.

The plan can therefore show an in-place change or replacement for index 1 (depending on the resource attributes) and destruction of index 2. Terraform is following positional identity; it does not understand that you intended to preserve the scheduler object by its name.

This is the main reason not to use `count` with a list of independently named resources that may be reordered or have items removed.

## 4. `for_each`: create keyed instances

Use `for_each` when each instance has a meaningful identity, such as service names, environment names, subnet roles, or account identifiers.

```hcl
variable "services" {
  type = set(string)
  default = [
    "api",
    "worker",
    "scheduler"
  ]
}

resource "aws_instance" "service" {
  for_each      = var.services
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}
```

The resource addresses are:

```text
aws_instance.service["api"]
aws_instance.service["worker"]
aws_instance.service["scheduler"]
```

For a map, use `each.key` for the key and `each.value` for the associated value:

```hcl
variable "services" {
  type = map(object({
    instance_type = string
  }))

  default = {
    api       = { instance_type = "t3.small" }
    worker    = { instance_type = "t3.medium" }
    scheduler = { instance_type = "t3.micro" }
  }
}

resource "aws_instance" "service" {
  for_each      = var.services
  ami           = var.ami_id
  instance_type = each.value.instance_type

  tags = {
    Name = each.key
  }
}
```

A map makes the identity and per-instance configuration explicit.

## 5. Address comparison

| Pattern | Example address | Identity |
|---|---|---|
| Single resource | `aws_instance.app` | Resource name |
| `count` | `aws_instance.app[0]` | Numeric index |
| `for_each` | `aws_instance.app["api"]` | Stable key |

Remember: the address is not the AWS instance ID. The address is Terraform's configuration/state identity; the provider's physical ID is an attribute recorded in state.

## 6. Implement and inspect the pattern

Use a disposable Terraform workspace and a safe AWS test account. Do not run this against production infrastructure.

Create a minimal `main.tf` using the map-based `for_each` example above, with a valid `ami_id` variable and AWS provider configuration from the earlier days.

Run:

```bash
terraform fmt
terraform init
terraform validate
terraform plan
```

Inspect the planned addresses. With the three keys above, expect:

```text
aws_instance.service["api"]
aws_instance.service["worker"]
aws_instance.service["scheduler"]
```

After applying only in the disposable environment, inspect state:

```bash
terraform state list
terraform state show 'aws_instance.service["api"]'
terraform state show 'aws_instance.service["worker"]'
terraform state show 'aws_instance.service["scheduler"]'
```

Quote addresses containing brackets and string keys so the shell does not interpret them.

Save and inspect a plan when you need a reviewable artifact:

```bash
terraform plan -out=tfplan
terraform show tfplan
```

## 7. Failure scenario — removing an item from a `count` list

Start with `["api", "worker", "scheduler"]` and apply in a disposable environment. Then remove `worker` from the list.

Run `terraform plan` and inspect the exact addresses. The remaining indexes are still 0 and 1; index 1 now receives the scheduler configuration, and index 2 is removed. Depending on which resource attributes change and the provider's replacement rules, index 1 may be updated or replaced.

Safe recovery and prevention:

1. Do not apply until you understand the plan.
2. Confirm which logical service each numeric address currently represents with `terraform state show`.
3. If this is a new design, migrate the configuration to `for_each` with meaningful keys before making further collection changes.
4. For already-managed resources, use Terraform's `moved` blocks or carefully reviewed `terraform state mv` operations to map old addresses to new addresses. Do not blindly change the resource expression and apply.
5. Run `terraform plan` again and confirm that existing objects are mapped to the intended keys, with no unintended destroy/create actions.
6. Apply only after reviewing the final plan.

## 8. Safe address migration: `count` to `for_each`

Changing from `count` to `for_each` changes addresses. Terraform may otherwise interpret the new addresses as new objects and the old addresses as objects to destroy.

For a controlled migration, a `moved` block can tell Terraform that an existing instance has a new address. For example, if the old index 0 is the API instance:

```hcl
moved {
  from = aws_instance.app[0]
  to   = aws_instance.service["api"]
}
```

Add one correctly mapped `moved` block per existing instance. The mapping must reflect what each old index actually represents; verify this against state and configuration rather than assuming list order.

Then run `terraform plan`. A correct address migration should show the address move without proposing replacement of the underlying object. Review all actions before applying. Keep migration blocks in configuration long enough for all relevant workspaces/state consumers to process the move, following your team's migration policy.

`terraform state mv` is an alternative for a carefully controlled state migration, but it directly changes state. Back up state, use the exact source and destination addresses, coordinate with the team, and immediately run a plan to verify the mapping. Never edit the state JSON manually.

## 9. When to choose each

Choose `count` when:
- You need a simple number of identical instances.
- You need a conditional singleton (`0` or `1`).
- Positional identity is acceptable and the collection will not be reordered or selectively edited.

Choose `for_each` when:
- Instances have meaningful names or keys.
- Items may be added or removed independently.
- Each instance has different settings.
- You need addresses that remain tied to logical identity rather than list position.

Do not choose `for_each` merely because it looks more advanced. Choose it when key-based identity reflects the real infrastructure model.

## 10. Common mistakes

- Removing a middle list item from a `count` resource without inspecting index changes.
- Assuming `count.index` is a permanent identity for a logical service.
- Using unstable or changing keys with `for_each`.
- Trying to use both `count` and `for_each` on the same resource block.
- Treating Terraform addresses as physical cloud IDs.
- Changing addresses and applying without a migration plan.
- Using `terraform state mv` without a backup and a follow-up plan.
- Forgetting to quote addresses containing brackets and string keys in shell commands.

## 11. Interview-ready mental model

**`count` = position-based identity; `for_each` = key-based identity.**

When a collection changes, ask: **Does this instance still represent the same logical object, and will Terraform's address still point to it?** For independently managed named resources, stable keys usually make `for_each` the safer choice.
