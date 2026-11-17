# DAY 17 — Build a Reusable AWS VPC Terraform Module

## 1. What we are building

A VPC is a logically isolated network in an AWS Region. A reusable VPC module should let the caller describe the network they need without repeating low-level AWS resource definitions in every environment.

Design principle:

- Root module: chooses CIDR ranges, subnet layout, tags, and whether NAT is enabled.
- Child module: creates the VPC resources and exposes useful IDs.
- Environment configuration: supplies different values for dev, QA, and production.

Today’s module creates:
- One VPC with DNS support and DNS hostnames enabled.
- Public and private subnets across the Availability Zones supplied by the caller.
- One Internet Gateway and a public route table.
- One private route table.
- An optional single NAT Gateway and private default route.
- Route-table associations and outputs for integration with other modules.

**Important design trade-off:** one NAT Gateway is simpler and cheaper for a learning/dev environment, but it is a single-AZ dependency and can create cross-AZ data-transfer charges. A production design commonly uses one NAT Gateway per AZ and routes each private subnet through its same-AZ NAT Gateway. Do not mistake this learning module for a complete production network baseline.

## 2. Suggested repository structure

~~~text
terraform-vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── terraform.tfvars
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── versions.tf
~~~

Keep the child module independent of environment-specific values. Do not hardcode account IDs, credentials, region names, CIDR ranges, or environment names inside it.

## 3. Child module: version and provider requirements

Create modules/vpc/versions.tf:

~~~hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0, < 7.0"
    }
  }
}
~~~

The child module declares its provider requirement, but normally inherits the configured AWS provider from the root module. It should not configure credentials or hardcode a provider region itself.

## 4. Child module: input contract

Create modules/vpc/variables.tf:

~~~hcl
variable "name" {
  description = "Name prefix applied to VPC resources."
  type        = string

  validation {
    condition     = length(trimspace(var.name)) > 0
    error_message = "name must not be empty."
  }
}

variable "vpc_cidr" {
  description = "IPv4 CIDR range for the VPC."
  type        = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid IPv4 CIDR block."
  }
}

variable "public_subnets" {
  description = "Public subnet definitions keyed by a stable subnet name."
  type = map(object({
    cidr_block        = string
    availability_zone = string
  }))
  default = {}
}

variable "private_subnets" {
  description = "Private subnet definitions keyed by a stable subnet name."
  type = map(object({
    cidr_block        = string
    availability_zone = string
  }))
  default = {}
}

variable "enable_nat_gateway" {
  description = "Create one NAT Gateway for private subnet egress. This is a single-AZ cost-conscious pattern."
  type        = bool
  default     = false
}

variable "tags" {
  description = "Additional tags applied to supported VPC resources."
  type        = map(string)
  default     = {}
}
~~~

Why use maps instead of lists?

Stable map keys produce stable resource addresses. Adding a subnet does not shift the identity of existing subnets as it can with positional count indexes.

## 5. Child module: create the network

Create modules/vpc/main.tf:

~~~hcl
locals {
  common_tags = merge(var.tags, {
    Name      = var.name
    ManagedBy = "Terraform"
  })
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(local.common_tags, {
    Name = var.name
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = local.common_tags
}

resource "aws_subnet" "public" {
  for_each = var.public_subnets

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.name}-public-${each.key}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  tags = merge(local.common_tags, {
    Name = "${var.name}-private-${each.key}"
    Tier = "private"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  tags = merge(local.common_tags, {
    Name = "${var.name}-public-rt"
  })
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id
  tags = merge(local.common_tags, {
    Name = "${var.name}-private-rt"
  })
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${var.name}-nat-eip"
  })
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = try(values(aws_subnet.public)[0].id, null)

  depends_on = [aws_internet_gateway.this, aws_route.public_internet]

  tags = merge(local.common_tags, {
    Name = "${var.name}-nat"
  })

  lifecycle {
    precondition {
      condition     = length(var.public_subnets) > 0
      error_message = "At least one public subnet is required when enable_nat_gateway is true."
    }
  }
}

resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? 1 : 0

  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.this[0].id
}

resource "aws_route_table_association" "private" {
  for_each = aws_subnet.private

  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}
~~~

The public route points to the Internet Gateway. A subnet is not public merely because it is named public: its associated route table must have a route to an Internet Gateway, and instances also need a public IPv4 address or another valid public addressing path.

The private route table receives a default route to NAT only when NAT is enabled. A NAT Gateway provides outbound connectivity for private-subnet resources; it does not allow unsolicited inbound connections from the internet.

The example uses the first public subnet for its single NAT Gateway. The precondition prevents NAT creation without a public subnet. For a stronger module contract, add input validation/tests for non-overlapping CIDRs, supported AZs, and private-subnet-to-NAT design.

## 6. Child module: expose useful outputs

Create modules/vpc/outputs.tf:

~~~hcl
output "vpc_id" {
  description = "ID of the created VPC."
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "IPv4 CIDR block of the VPC."
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Map of public subnet keys to subnet IDs."
  value       = { for key, subnet in aws_subnet.public : key => subnet.id }
}

output "private_subnet_ids" {
  description = "Map of private subnet keys to subnet IDs."
  value       = { for key, subnet in aws_subnet.private : key => subnet.id }
}

output "public_route_table_id" {
  description = "ID of the public route table."
  value       = aws_route_table.public.id
}

output "private_route_table_id" {
  description = "ID of the private route table."
  value       = aws_route_table.private.id
}

output "nat_gateway_id" {
  description = "NAT Gateway ID, or null when NAT is disabled."
  value       = try(aws_nat_gateway.this[0].id, null)
}
~~~

Outputs form the module's public interface. A downstream security-group, load-balancer, or compute module should consume these outputs rather than reaching into the child module's internal resource addresses.

## 7. Root module: call the reusable VPC module

Root versions.tf:

~~~hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0, < 7.0"
    }
  }
}
~~~

Root providers.tf:

~~~hcl
variable "aws_region" {
  type        = string
  description = "AWS deployment region."
  default     = "ap-south-1"
}

provider "aws" {
  region = var.aws_region
}
~~~

Root main.tf:

~~~hcl
module "network" {
  source = "./modules/vpc"

  name      = "orders-dev"
  vpc_cidr  = "10.40.0.0/16"

  public_subnets = {
    public_a = {
      cidr_block        = "10.40.1.0/24"
      availability_zone = "ap-south-1a"
    }
    public_b = {
      cidr_block        = "10.40.2.0/24"
      availability_zone = "ap-south-1b"
    }
  }

  private_subnets = {
    private_a = {
      cidr_block        = "10.40.11.0/24"
      availability_zone = "ap-south-1a"
    }
    private_b = {
      cidr_block        = "10.40.12.0/24"
      availability_zone = "ap-south-1b"
    }
  }

  enable_nat_gateway = false

  tags = {
    Environment = "dev"
    Project     = "orders"
    Owner       = "platform"
  }
}
~~~

Root outputs.tf:

~~~hcl
output "vpc_id" {
  value = module.network.vpc_id
}

output "public_subnet_ids" {
  value = module.network.public_subnet_ids
}

output "private_subnet_ids" {
  value = module.network.private_subnet_ids
}
~~~

The root module calls the child module once. A QA or production root configuration can call the same module with different CIDRs, subnet maps, tags, and NAT settings.

## 8. Initialize, validate, and inspect the plan

Run these commands from the root directory:

~~~bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan
~~~

Do not apply until you have checked:
- The active AWS account and region.
- The VPC and subnet CIDRs do not overlap with existing connected networks.
- The intended Availability Zones exist in the selected Region.
- NAT is disabled or enabled intentionally; NAT Gateway and public IPv4 resources can incur charges.
- No unexpected destroy or replacement action appears.

For this example with NAT disabled, the expected managed resource address patterns are:

~~~text
module.network.aws_vpc.this
module.network.aws_internet_gateway.this
module.network.aws_subnet.public["public_a"]
module.network.aws_subnet.public["public_b"]
module.network.aws_subnet.private["private_a"]
module.network.aws_subnet.private["private_b"]
module.network.aws_route_table.public
module.network.aws_route.public_internet
module.network.aws_route_table_association.public["public_a"]
module.network.aws_route_table_association.public["public_b"]
module.network.aws_route_table.private
module.network.aws_route_table_association.private["private_a"]
module.network.aws_route_table_association.private["private_b"]
~~~

When NAT is enabled, these additional addresses appear:

~~~text
module.network.aws_eip.nat[0]
module.network.aws_nat_gateway.this[0]
module.network.aws_route.private_nat[0]
~~~

The exact addresses depend on the module block name, resource labels, and for_each keys. If the caller renames module.network, the module-qualified addresses change and require a state migration strategy rather than casual renaming.

After applying in a disposable AWS account, inspect:

~~~bash
terraform state list
terraform state show 'module.network.aws_vpc.this'
terraform state show 'module.network.aws_subnet.public["public_a"]'
terraform output vpc_id
terraform output public_subnet_ids
terraform output private_subnet_ids
~~~

Record both Terraform addresses and physical AWS IDs. The address identifies the object in Terraform state; the ID identifies the object in AWS.

## 9. Failure scenario — NAT requested without a public subnet

Scenario: a caller sets enable_nat_gateway = true but passes an empty public_subnets map.

Expected behavior: the NAT Gateway resource precondition should fail before Terraform creates the NAT Gateway. Do not remove the precondition to force the plan through.

Safe recovery:
1. Read the diagnostic and identify module.network.aws_nat_gateway.this[0].
2. Confirm the selected environment, region, and subnet inputs.
3. Add at least one valid public subnet, or disable NAT if private-subnet internet egress is not required.
4. Run fmt, validate, and plan again.
5. Confirm the public subnet has a route to the Internet Gateway.
6. Review NAT Gateway cost and AZ placement before apply.

Other realistic failure: AWS rejects a subnet because its CIDR overlaps an existing subnet or is outside the VPC CIDR. Fix the input CIDR design; do not work around the error by deleting unrelated state.

## 10. Drift scenario — someone changes a subnet tag manually

Scenario: the AWS console changes the Name tag on the subnet represented by:

~~~text
module.network.aws_subnet.private["private_a"]
~~~

Recovery:
1. Run terraform plan and identify the exact address and tag difference.
2. Confirm whether Terraform or an approved external process owns the tag.
3. If Terraform owns it, restore the intended tag in module inputs/locals and plan again.
4. If external ownership is intentional, define a narrow ownership boundary; do not ignore every tag to hide drift.
5. Apply only after review, then run a fresh plan and inspect state.

Do not use terraform state rm to fix ordinary drift. State removal does not repair the AWS resource and can cause Terraform to propose creating another object.

## 11. Production design considerations

Before reusing a VPC module in production, decide and document:

- NAT strategy: one NAT per AZ for resilience, or a deliberate lower-cost alternative.
- Egress control: NAT alone is not a firewall or full egress security solution.
- CIDR planning: account for peering, Transit Gateway, VPN, on-premises networks, and future growth.
- Availability Zones: avoid hardcoding assumptions that do not hold in every Region/account.
- Endpoints: add Gateway endpoints for S3/DynamoDB and interface endpoints where private service access is needed.
- Flow logs: centralize VPC Flow Logs with retention and access controls.
- DNS: define resolver, private hosted-zone, and hybrid DNS requirements.
- Security: use separate security groups, least-privilege rules, and controlled ingress/egress.
- State: use remote encrypted state, locking, environment isolation, and restricted access.
- Tests: validate inputs, subnet mapping, optional NAT behavior, tags, and route intent with terraform test; use sandbox integration tests for real AWS behavior.

Do not turn every possible network feature into a mandatory input on day one. Keep the module interface small, add capabilities when a real consumer needs them, and version changes deliberately.

## 12. Common mistakes

- Creating a subnet and assuming it is public without checking its route table.
- Forgetting that a public subnet route to an Internet Gateway does not automatically assign a public IP to every instance.
- Routing private subnets directly to an Internet Gateway.
- Creating NAT without a public subnet.
- Assuming one NAT Gateway across multiple AZs is highly available.
- Using count indexes for subnet identity when stable for_each keys are more suitable.
- Hardcoding environment-specific CIDRs or account details inside the child module.
- Exposing internal resources instead of a small stable set of outputs.
- Renaming a module block without considering state addresses.
- Applying a plan without checking account, region, CIDR overlap, and NAT cost.
- Treating terraform validate or a mock test as proof that AWS will accept the deployment.

## 13. Interview-ready mental model

**A reusable VPC module is a contract: inputs describe the desired network, resources implement it, outputs expose stable integration points, and the root module supplies environment-specific choices.**

A public subnet reaches the internet through an Internet Gateway route; a private subnet uses NAT for outbound IPv4 internet access when required. Terraform module-qualified addresses and stable for_each keys make the infrastructure inspectable and safer to evolve.
