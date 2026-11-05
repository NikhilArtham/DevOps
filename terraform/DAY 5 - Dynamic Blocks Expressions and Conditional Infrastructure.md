# DAY 5 — Dynamic Blocks, Expressions and Conditional Infrastructure

## 1. Why these features matter

Reusable Terraform modules need to handle different environments and optional configuration without copying entire resource blocks.

- **Expressions** calculate values from variables, locals, resources, and data sources.
- **Conditional expressions** select a value based on a condition.
- **Dynamic blocks** generate repeated nested configuration blocks from a collection.

A dynamic block does not create separate Terraform resources. It generates nested blocks inside one resource. This differs from resource-level `count` or `for_each`.

## 2. Expressions

### Conditional expression

Syntax: `condition ? true_value : false_value`.

```hcl
variable "environment" {
  type = string
}

locals {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
}
```

Both result branches should have compatible types.

### For expressions

A `for` expression transforms a collection:

```hcl
variable "service_names" {
  type    = list(string)
  default = ["api", "worker", "scheduler"]
}

locals {
  service_tags = {
    for name in var.service_names : name => upper(name)
  }

  production_services = [
    for name in var.service_names : name
    if name != "scheduler"
  ]
}
```

### Useful functions

- `length(value)`: number of elements or characters.
- `contains(collection, value)`: checks membership.
- `lookup(map, key, default)`: reads a map value with a fallback.
- `merge(map1, map2)`: combines maps; later values win for duplicate keys.
- `try(expression, fallback)`: returns the first expression that does not error.
- `coalesce(value1, value2, ...)`: returns the first non-null, non-empty string value.

Avoid using `try` or fallback values to silently conceal malformed input. Prefer clear validation where possible.

## 3. Conditional infrastructure with count

Use a conditional count for an optional singleton resource:

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name          = "app-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}
```

When enabled, the address is `aws_cloudwatch_metric_alarm.high_cpu[0]`. When switched from enabled to disabled, Terraform plans to destroy the previously managed alarm.

Use `count` for a conditional singleton. For multiple independently named optional resources, use `for_each` with meaningful keys.

## 4. Dynamic block syntax

```hcl
dynamic "BLOCK_TYPE" {
  for_each = COLLECTION

  content {
    # arguments for one generated nested block
  }
}
```

- The label must match a nested block supported by the resource's provider schema.
- `for_each` is the collection to iterate over.
- `content` defines one generated nested block.
- The default iterator name is the dynamic block label. Use `iterator` to choose a clearer name.
- Dynamic blocks cannot generate meta-argument blocks such as `lifecycle` or `provisioner`.

## 5. AWS example — generate security-group ingress rules

```hcl
variable "ingress_rules" {
  description = "Approved ingress rules for the application"
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))

  default = [
    {
      description = "HTTP from corporate network"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
    },
    {
      description = "HTTPS from corporate network"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
    }
  ]

  validation {
    condition = alltrue([
      for rule in var.ingress_rules :
      rule.from_port >= 0 && rule.from_port <= 65535 &&
      rule.to_port >= rule.from_port && rule.to_port <= 65535
    ])
    error_message = "Ports must be valid and to_port must be greater than or equal to from_port."
  }
}

resource "aws_security_group" "app" {
  name        = "app-security-group"
  description = "Application ingress rules"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    iterator = rule

    content {
      description = rule.value.description
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
    }
  }

  tags = {
    Name = "app-security-group"
  }
}
```

This creates one `aws_security_group.app` resource with two nested ingress blocks. It does not create two security-group resources. The resource address is simply `aws_security_group.app`.

Security note: validate CIDR ranges and avoid exposing SSH/RDP or other management ports to untrusted networks. Add only approved sources and required ports.

## 6. Optional nested blocks

A dynamic block can generate zero or one nested blocks. This is a reusable pattern fragment; put it only inside a resource whose provider schema supports the named nested block and arguments.

```hcl
variable "health_check" {
  type = object({
    path                = string
    healthy_threshold   = number
    unhealthy_threshold = number
  })
  default = null
}

dynamic "health_check" {
  for_each = var.health_check == null ? [] : [var.health_check]

  content {
    path                = health_check.value.path
    healthy_threshold   = health_check.value.healthy_threshold
    unhealthy_threshold = health_check.value.unhealthy_threshold
  }
}
```

An empty collection generates no block; a one-element collection generates one block. Do not assume every resource supports a nested block with this exact schema.

## 7. Which feature should you use?

| Requirement | Feature | Result |
|---|---|---|
| Select one value | Conditional expression | One value |
| Transform/filter a collection | `for` expression | New collection |
| Create or omit resources | Resource `count` / `for_each` | Resource instances |
| Repeat nested settings | `dynamic` block | Nested blocks inside one resource |

Do not use a dynamic block when a normal argument or resource-level `for_each` is the clearer solution.

## 8. Commands to practice

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform console
terraform state list
terraform state show aws_security_group.app
terraform show
```

- `terraform fmt`: formats HCL files.
- `terraform init`: initializes provider and module dependencies.
- `terraform validate`: checks syntax and internal consistency; it does not guarantee the cloud API will accept every request.
- `terraform plan`: evaluates expressions and displays proposed changes.
- `terraform console`: interactively tests expressions.
- `terraform state list`: lists tracked resource addresses.
- `terraform state show <address>`: displays state for one address.
- `terraform show`: displays current state or a saved plan.

Try these in `terraform console`:

```hcl
var.environment == "prod" ? "t3.large" : "t3.micro"
[for port in [80, 443, 8080] : port if port != 8080]
merge({ Environment = "dev" }, { Owner = "platform" })
```

## 9. Realistic failure scenario — malformed ingress input

A shared security-group module accepts ingress rules from environment-specific tfvars files. A developer adds a rule but misspells `from_port` as `form_port`, or supplies a string where the object schema expects a number.

Possible results:
- Terraform reports an object type or unsupported attribute error during validation/plan evaluation.
- A missing field prevents the dynamic content from being evaluated.
- A valid but overly broad CIDR or unnecessary port can still create an insecure rule.

Safe recovery:

1. Stop before applying. Identify the module and resource address, for example `aws_security_group.app`.
2. Inspect the variable's declared object type and the exact value in the relevant tfvars file or module call.
3. Check every key against the schema: `description`, `from_port`, `to_port`, `protocol`, and `cidr_blocks`.
4. Run `terraform fmt` and `terraform validate`.
5. Run `terraform plan` and inspect every generated ingress rule, including ports and CIDRs.
6. Correct the input at its source. Do not weaken the type constraint or add broad fallback values just to suppress the error.
7. Verify management ports are not exposed to untrusted networks and the plan changes only intended rules.
8. Apply only after review and approval.

If an unsafe rule was already applied, follow the incident/change process, correct the configuration, plan the least disruptive remediation, and verify the live security-group rules afterward.

## 10. AWS and Kubernetes relevance

AWS:
- Security-group ingress/egress nested rules.
- Optional CloudWatch alarms controlled by variables.
- Environment-specific settings where the provider schema supports nested blocks.
- Reusable modules accepting lists/maps of network rules, subnet settings, and policy statements.

Kubernetes:
- Conditional namespace, service-account, or deployment resources using `count` or `for_each`.
- Expressions to derive labels, annotations, and replica counts.
- Dynamic nested blocks only when the selected Kubernetes provider resource exposes repeatable nested blocks. Arbitrary Kubernetes YAML/JSON resources often use expressions and collection transformations instead.

## 11. Common mistakes

- Confusing dynamic nested blocks with resource-level `for_each`.
- Assuming dynamic blocks create separately addressable resources.
- Using a block label unsupported by the resource schema.
- Using the wrong iterator name or referencing the wrong value.
- Forgetting that changing a resource count from one to zero destroys the managed instance.
- Using `try` to hide invalid inputs.
- Assuming `terraform validate` guarantees API success.
- Generating security rules without validating ports, protocols, and CIDR exposure.
- Using dynamic blocks for simple fixed configuration that is clearer as ordinary HCL.

## 12. Interview-ready mental model

**Expression calculates a value → condition chooses a path → resource meta-argument decides whether/how many resources exist → dynamic block repeats nested configuration inside one resource.**

When a dynamic block fails, inspect the provider schema, input object shape, iterator values, and exact plan before changing infrastructure.
