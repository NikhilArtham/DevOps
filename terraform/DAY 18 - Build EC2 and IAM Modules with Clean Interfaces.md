# DAY 18 — Build EC2 and IAM Modules with Clean Interfaces

## 1. Objective and module boundaries

Build two independent child modules and connect them from the root module:

- IAM module: creates an EC2-trusted role, attaches Systems Manager permissions, and creates an instance profile.
- EC2 module: launches an EC2 instance using an AMI, subnet, security groups, and the supplied instance-profile name.

Keep ownership clear:

| Concern | Owner |
|---|---|
| IAM trust relationship and role permissions | IAM module |
| Instance profile | IAM module |
| AMI, instance type, subnet and security groups | EC2 module caller |
| EC2 instance and root EBS volume settings | EC2 module |
| VPC, subnet and security-group creation | Network/security modules or root composition |
| Region, credentials and provider configuration | Root module |

The EC2 module must not silently create broad IAM permissions. The IAM module must not hardcode an EC2 instance ID, subnet, account, or region.

We will use Systems Manager Session Manager rather than opening inbound SSH. The example assumes the selected AMI includes a compatible SSM Agent. Verify the agent for custom images before deploying.

## 2. Suggested directory structure

~~~text
terraform-ec2/
├── versions.tf
├── providers.tf
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    ├── iam-ec2/
    │   ├── versions.tf
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2/
        ├── versions.tf
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
~~~

Each module has one responsibility and a small input/output contract. The root module composes the modules and passes outputs from one module into another.

## 3. IAM child module — trust policy, role and instance profile

### 3.1 Provider requirement

Create modules/iam-ec2/versions.tf:

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

### 3.2 Inputs

Create modules/iam-ec2/variables.tf:

~~~hcl
variable "name" {
  description = "Name prefix for the EC2 role and instance profile."
  type        = string

  validation {
    condition     = length(trimspace(var.name)) > 0
    error_message = "name must not be empty."
  }
}

variable "tags" {
  description = "Tags applied to IAM resources where supported."
  type        = map(string)
  default     = {}
}
~~~

### 3.3 Resources

Create modules/iam-ec2/main.tf:

~~~hcl
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "this" {
  name               = "${var.name}-ec2-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
  tags               = var.tags
}

resource "aws_iam_role_policy_attachment" "ssm_core" {
  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "this" {
  name = "${var.name}-instance-profile"
  role = aws_iam_role.this.name
  tags = var.tags
}
~~~

Understand the three separate objects:

1. IAM role: an identity with permissions and a trust policy.
2. Trust policy: allows the EC2 service to assume the role.
3. Instance profile: the EC2 attachment container through which the instance receives the role.

The AWS-managed AmazonSSMManagedInstanceCore policy grants the permissions commonly required for SSM managed-instance communication. It is a starting point for this exercise, not a reason to attach AdministratorAccess. Review the current AWS policy and your organization's approved permission baseline before production use.

### 3.4 Outputs

Create modules/iam-ec2/outputs.tf:

~~~hcl
output "role_name" {
  description = "Name of the EC2 IAM role."
  value       = aws_iam_role.this.name
}

output "role_arn" {
  description = "ARN of the EC2 IAM role."
  value       = aws_iam_role.this.arn
}

output "instance_profile_name" {
  description = "Name to pass to the EC2 module."
  value       = aws_iam_instance_profile.this.name
}

output "instance_profile_arn" {
  description = "ARN of the EC2 instance profile."
  value       = aws_iam_instance_profile.this.arn
}
~~~

The EC2 module needs the profile name. Consumers that need the role ARN can use the separate role output. Do not expose every internal resource attribute without a consumer requirement.

## 4. EC2 child module — compute without IAM ownership

### 4.1 Provider requirement

Create modules/ec2/versions.tf:

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

### 4.2 Inputs

Create modules/ec2/variables.tf:

~~~hcl
variable "name" {
  description = "Name tag for the EC2 instance."
  type        = string
}

variable "ami_id" {
  description = "Approved AMI ID for the target Region."
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type."
  type        = string
  default     = "t3.micro"
}

variable "subnet_id" {
  description = "Subnet ID where the instance will be launched."
  type        = string
}

variable "security_group_ids" {
  description = "Security groups created and managed outside this module."
  type        = set(string)
}

variable "instance_profile_name" {
  description = "Existing IAM instance profile name to attach."
  type        = string
}

variable "root_volume_size" {
  description = "Root EBS volume size in GiB."
  type        = number
  default     = 20

  validation {
    condition     = var.root_volume_size >= 8
    error_message = "root_volume_size must be at least 8 GiB."
  }
}

variable "tags" {
  description = "Additional instance tags."
  type        = map(string)
  default     = {}
}
~~~

Notice what is deliberately absent:
- No IAM role creation.
- No security-group creation.
- No hardcoded subnet or VPC.
- No embedded AWS credentials.
- No SSH key or inbound SSH rule requirement.

These are caller-owned concerns. If a workload genuinely needs SSH, design a controlled access path rather than adding 0.0.0.0/0 ingress by default.

### 4.3 EC2 resource

Create modules/ec2/main.tf:

~~~hcl
resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids
  iam_instance_profile   = var.instance_profile_name

  associate_public_ip_address = false
  monitoring                  = true

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  root_block_device {
    volume_size           = var.root_volume_size
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }

  tags = merge(var.tags, {
    Name      = var.name
    ManagedBy = "Terraform"
  })
}
~~~

Security choices:
- IMDSv2 is required with http_tokens = "required".
- The root EBS volume is encrypted and uses gp3.
- No public IPv4 address is automatically assigned.
- The module accepts security groups but does not create broad ingress rules.
- Detailed monitoring is enabled; assess its additional CloudWatch cost before production use.

### 4.4 Outputs

Create modules/ec2/outputs.tf:

~~~hcl
output "instance_id" {
  description = "EC2 instance ID."
  value       = aws_instance.this.id
}

output "instance_arn" {
  description = "EC2 instance ARN."
  value       = aws_instance.this.arn
}

output "private_ip" {
  description = "Private IPv4 address."
  value       = aws_instance.this.private_ip
}

output "availability_zone" {
  description = "Availability Zone where EC2 was launched."
  value       = aws_instance.this.availability_zone
}
~~~

## 5. Root module — compose IAM and EC2

The root module owns provider configuration and connects the two child modules. The subnet and security-group IDs below are inputs supplied by the network/security layer. They can come from the VPC module from DAY 17 and a separately managed security-group resource/module.

Example root main.tf:

~~~hcl
module "instance_role" {
  source = "./modules/iam-ec2"

  name = "orders-dev"

  tags = {
    Environment = "dev"
    Project     = "orders"
    ManagedBy   = "Terraform"
  }
}

module "app_server" {
  source = "./modules/ec2"

  name                  = "orders-dev-app"
  ami_id                = var.ami_id
  instance_type         = "t3.micro"
  subnet_id             = var.private_subnet_id
  security_group_ids    = var.app_security_group_ids
  instance_profile_name = module.instance_role.instance_profile_name
  root_volume_size      = 20

  tags = {
    Environment = "dev"
    Project     = "orders"
  }

  depends_on = [module.instance_role]
}
~~~

The reference to module.instance_role.instance_profile_name creates an implicit dependency on the instance profile. The explicit module-level depends_on also waits for all resources in the IAM module, including the SSM policy attachment, before creating the EC2 instance. Use this deliberate ordering because an instance can boot before the role policy attachment is ready if Terraform only waits for the profile.

Example root variables.tf:

~~~hcl
variable "ami_id" {
  description = "Approved AMI ID in the configured AWS Region."
  type        = string
}

variable "private_subnet_id" {
  description = "Private subnet ID from the network layer."
  type        = string
}

variable "app_security_group_ids" {
  description = "Security groups allowing only required application and egress traffic."
  type        = set(string)
}
~~~

Example root outputs.tf:

~~~hcl
output "instance_id" {
  value = module.app_server.instance_id
}

output "instance_private_ip" {
  value = module.app_server.private_ip
}

output "instance_profile_name" {
  value = module.instance_role.instance_profile_name
}
~~~

If using the DAY 17 VPC module in the same root configuration, a subnet value can be wired from an output such as module.network.private_subnet_ids["private_a"]. Do not hardcode an AWS resource ID inside the reusable EC2 module.

## 6. Trace the EC2 boot and access path

Think of the instance as a chain of dependent layers:

~~~text
Terraform root module
  |
  v
AWS EC2 RunInstances API
  |
  v
Subnet + route table + security groups
  |
  v
AMI boot + EBS root volume + cloud-init
  |
  v
EC2 instance profile attached
  |
  v
IMDSv2 provides temporary role credentials
  |
  v
SSM Agent uses role permissions
  |
  v
Outbound HTTPS to Systems Manager endpoints
  |
  v
Instance registers as a managed node
  |
  v
Session Manager access
~~~

A failure at one layer can look like an application or operating-system issue. Diagnose from the bottom of the chain rather than randomly changing IAM and networking.

### Layer 1 — EC2 launch and placement

Check:
- Did RunInstances succeed?
- Is the instance in the intended VPC, subnet and Availability Zone?
- Is the root EBS volume attached and healthy?
- Is the instance status check passing?

Useful commands from an authorized operator workstation:

~~~bash
aws ec2 describe-instances   --instance-ids i-xxxxxxxxxxxxxxxxx   --query 'Reservations[].Instances[].{State:State.Name,AZ:Placement.AvailabilityZone,Subnet:SubnetId,PrivateIP:PrivateIpAddress,IamProfile:IamInstanceProfile.Arn}'
~~~

The query shows state, placement, subnet, private IP and attached profile ARN without printing credentials.

### Layer 2 — Instance metadata and IAM identity

IMDSv2 is enabled and required in the EC2 module. From a shell on the instance, request a metadata token:

~~~bash
TOKEN=$(curl -sS -X PUT "http://169.254.169.254/latest/api/token"   -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
~~~

Read safe instance metadata:

~~~bash
curl -sS -H "X-aws-ec2-metadata-token: $TOKEN"   http://169.254.169.254/latest/meta-data/instance-id

curl -sS -H "X-aws-ec2-metadata-token: $TOKEN"   http://169.254.169.254/latest/meta-data/iam/info
~~~

The IAM metadata response can help confirm an instance profile is associated. Do not retrieve or print the metadata credentials endpoint in logs, tickets, terminal recordings, or chat.

Check the role outside the instance:

~~~bash
aws iam get-instance-profile   --instance-profile-name orders-dev-instance-profile

aws iam list-attached-role-policies   --role-name orders-dev-ec2-role
~~~

The role trust policy must allow the EC2 service principal, ec2.amazonaws.com, to assume the role. The instance profile must contain that role.

### Layer 3 — Operating system and SSM Agent

On a Linux instance:

~~~bash
sudo systemctl status amazon-ssm-agent
sudo journalctl -u amazon-ssm-agent --since "30 minutes ago"
sudo tail -n 100 /var/log/amazon/ssm/amazon-ssm-agent.log
cloud-init status --long
sudo journalctl -u cloud-final --since "30 minutes ago"
~~~

Use these to distinguish:
- Agent not installed.
- Agent installed but stopped or repeatedly crashing.
- Agent running but unable to obtain role credentials.
- Agent running but unable to reach the SSM service endpoints.

The exact service/package name and log path can vary by operating system and image. Confirm the selected AMI's SSM Agent installation and service details.

### Layer 4 — Network path to Systems Manager

The SSM Agent needs outbound HTTPS connectivity to the regional Systems Manager services. For a private subnet, provide a supported egress path such as:
- NAT Gateway with valid routes and egress controls; or
- Interface VPC endpoints for the required Systems Manager services, with private DNS and endpoint security-group rules.

Common endpoints to evaluate include ssm and ssmmessages. ec2messages may be relevant for older agent/Region compatibility. Confirm the current endpoint requirements for the selected Region and agent version.

Check:
- Private subnet route table.
- NAT Gateway health and route if NAT is used.
- VPC DNS support and DNS hostnames.
- Endpoint private DNS configuration.
- Endpoint security group allowing TCP 443 from the instance.
- Instance security-group egress allowing the required HTTPS path.
- Network ACL return traffic.

No inbound internet rule is required for Session Manager. The agent establishes outbound connections.

From an operator workstation:

~~~bash
aws ssm describe-instance-information   --filters "Key=InstanceIds,Values=i-xxxxxxxxxxxxxxxxx"
~~~

If the instance is absent or not online, correlate the IAM, agent and network checks above. Do not open inbound SSH as a shortcut to an SSM registration failure.

## 7. Inspect Terraform plan and exact resource addresses

Run from the root module:

~~~bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan
~~~

Expected managed resource address patterns:

~~~text
module.instance_role.aws_iam_role.this
module.instance_role.aws_iam_role_policy_attachment.ssm_core
module.instance_role.aws_iam_instance_profile.this
module.app_server.aws_instance.this
~~~

The IAM policy document is a data source, so its address is:

~~~text
module.instance_role.data.aws_iam_policy_document.ec2_assume_role
~~~

Inspect state after a reviewed sandbox apply:

~~~bash
terraform state list
terraform state show module.instance_role.aws_iam_role.this
terraform state show module.instance_role.aws_iam_instance_profile.this
terraform state show module.instance_role.aws_iam_role_policy_attachment.ssm_core
terraform state show module.app_server.aws_instance.this
terraform output instance_id
terraform output instance_private_ip
~~~

Record:
- Terraform address.
- Physical AWS ID or ARN.
- IAM role name and attached policy.
- Instance-profile name.
- EC2 instance ID, subnet, private IP and security groups.

Do not publish state files or saved plan artifacts. They can contain sensitive infrastructure details.

## 8. Failure scenario — EC2 launches but is not online in Systems Manager

Scenario: Terraform successfully creates the EC2 instance, but the instance does not appear as an online managed node in Systems Manager.

A likely IAM-layer cause is that the role is attached to the instance profile but does not have the required SSM managed-instance permissions.

Diagnose in order:

1. Confirm the EC2 instance is running and status checks pass.
2. Confirm the expected instance profile is attached using describe-instances.
3. Confirm the profile contains the expected role.
4. Confirm the trust policy allows ec2.amazonaws.com to assume the role.
5. Confirm AmazonSSMManagedInstanceCore is attached to the role.
6. Check IMDSv2 IAM metadata from the instance without exposing credential values.
7. Check amazon-ssm-agent service status and logs.
8. If IAM and agent checks pass, investigate DNS, route tables, NAT/VPC endpoints, endpoint security groups and outbound TCP 443.

Recovery:
- Fix the missing role attachment or trust relationship in the IAM module.
- Run terraform plan and verify that only the intended IAM change is proposed.
- Apply through the approved workflow.
- Allow the agent to retry registration, then verify with describe-instance-information.
- If still offline, continue with the agent and network layers. Do not assume an IAM fix resolves a separate network failure.

A successful EC2 launch proves that the EC2 API accepted the launch request. It does not prove that the instance can assume its role, reach SSM, or complete application bootstrap.

## 9. Common mistakes

- Confusing an IAM role with an instance profile.
- Using a trust policy that does not allow the EC2 service principal.
- Attaching permissions to a user instead of the role used by the instance.
- Assuming an instance profile automatically provides network connectivity.
- Attaching AdministratorAccess to solve an SSM issue.
- Creating IAM resources inside every EC2 module and duplicating roles unnecessarily.
- Allowing public SSH from 0.0.0.0/0 when Session Manager is the intended access path.
- Forgetting SSM Agent installation or service health in custom AMIs.
- Forgetting NAT or the required VPC endpoints for private instances.
- Assuming a successful EC2 launch means cloud-init and application bootstrap succeeded.
- Changing state manually before checking the live AWS instance and IAM attachments.
- Printing IMDS credentials or storing them in logs.

## 10. Production design considerations

Before standardizing these modules:
- Decide whether IAM roles are per workload, per application, or shared by a controlled instance fleet.
- Attach only the permissions the workload needs; keep SSM administration and application permissions deliberate.
- Review the AWS-managed SSM policy and use a reviewed customer-managed policy if organizational controls require narrower permissions.
- Use approved, patched AMIs and verify SSM Agent availability.
- Keep instances private where possible and use Session Manager for administrative access.
- Design VPC endpoints/NAT, DNS, egress and endpoint security groups explicitly.
- Require IMDSv2 and review metadata hop-limit requirements for containers or proxy use cases.
- Encrypt EBS volumes with the approved KMS key and define backup/retention separately.
- Separate instance provisioning from application deployment when their lifecycles differ.
- Use remote state, locking, short-lived CI credentials and reviewed plans.
- Add tests for module input validation, IAM trust policy, required policy attachments, metadata settings, encrypted storage and expected outputs.

## 11. Interview-ready mental model

**IAM role = permissions and trust; instance profile = attachment mechanism; EC2 module = compute configuration.**

For SSM access, trace: EC2 launch → instance profile → IMDSv2 role credentials → SSM Agent → outbound HTTPS and service endpoints → managed-node registration. A running instance is not automatically a manageable instance.
