# AWS

AWS notes for DevOps work, written beginner-first and focused on architecture, troubleshooting, production scenarios, Terraform implementation, and interview readiness.

## Study Structure

- `DAY X - topic.md` → learning notes only
- `LABS.md` → all practical AWS labs
- `INTERVIEW QUESTIONS.md` → all AWS interview questions
- `README.md` → AWS learning path and progress

**Do not create separate per-day LABS or INTERVIEW QUESTIONS files.**

DAY numbering starts at `DAY 1` independently for AWS.

## Progress

- [DAY 1 - AWS VPC Architecture: CIDR, Subnets and Traffic Flow](DAY%201%20-%20AWS%20VPC%20Architecture%20CIDR%20Subnets%20and%20Traffic%20Flow.md) — VPC fundamentals, CIDR planning, subnet design, route tables, public/private subnets, Internet Gateway, NAT Gateway, Security Groups, NACLs, traffic flow, AWS CLI inspection, EKS connection, and a realistic private-subnet connectivity failure.
- [DAY 2 - Route Tables, IGW, NAT Gateway and VPC Endpoints](DAY%202%20-%20Route%20Tables%20IGW%20NAT%20Gateway%20and%20VPC%20Endpoints.md) — route matching, Internet Gateway, NAT Gateway, gateway/interface VPC endpoints, AWS CLI inspection, private AWS service access, EKS connection, and a realistic route-table failure.
- [DAY 3 - Security Groups vs NACLs: Stateful vs Stateless Troubleshooting](DAY%203%20-%20Security%20Groups%20vs%20NACLs%20Stateful%20vs%20Stateless%20Troubleshooting.md) — stateful vs stateless filtering, SG/NACL failure paths, AWS CLI inspection, Terraform resource addresses, plan/state inspection, drift detection, safe recovery, EKS connection, and a realistic NACL return-traffic failure.
- [DAY 4 - ALB vs NLB: Listeners, Target Health and Routing](DAY%204%20-%20ALB%20vs%20NLB%20Listeners%20Target%20Health%20and%20Routing.md) — ALB vs NLB, Layer 7 vs Layer 4, listeners, ALB listener rules, target groups, target health, health-check troubleshooting, AWS CLI inspection, EKS connection, common 502/503/504 failure paths, and a realistic unhealthy-target incident.
- [DAY 5 - Route 53: Records, Health Checks and DNS Troubleshooting](DAY%205%20-%20Route%2053%20Records%20Health%20Checks%20and%20DNS%20Troubleshooting.md) — Route 53 records, public/private hosted zones, alias records, TTL and caching, DNS routing policies, health checks, `dig`/`nslookup`, end-to-end DNS troubleshooting with `ip`, `ss`, `ping`, `traceroute`, `nc`, `openssl`, and `curl`, ALB/EKS integration, and an intermittent DNS-based production failure.
- [DAY 6 - EC2 Boot, User-Data, Metadata and System Troubleshooting](DAY%206%20-%20EC2%20Boot%20User-data%20Metadata%20and%20System%20Troubleshooting.md) — EC2 boot flow, instance/status checks, user-data and cloud-init, IMDSv2, SSH vs SSM access, system logs, application/listening-port checks, AWS CLI inspection, and a realistic bootstrap failure causing an unhealthy ALB target.
- [DAY 7 - Systems Manager: Session Manager, Run Command and Patch](DAY%207%20-%20AWS%20Systems%20Manager%20Session%20Manager%20Run%20Command%20and%20Patch.md) — managed-node prerequisites, SSM Agent, IAM and network dependencies, Session Manager, port forwarding, Run Command, SSM documents, Patch Manager, patch compliance, AWS CLI operations, troubleshooting, EKS relevance, and realistic SSM/session/patch failure scenarios.
- [DAY 8 - IAM Policy Evaluation, Explicit Deny and Least Privilege](DAY%208%20-%20IAM%20Policy%20Evaluation%20Explicit%20Deny%20and%20Least%20Privilege.md) — identity/resource policies, principal-action-resource-condition analysis, explicit denies, permissions boundaries, session policies, SCPs, IAM Policy Simulator, STS caller identification, AccessDenied troubleshooting, and least-privilege remediation.
- [DAY 9 - Cross-account IAM Roles and Trust Policies](DAY%209%20-%20Cross-account%20IAM%20Roles%20and%20Trust%20Policies.md) — cross-account AssumeRole flow, trust vs permissions policies, source-side `sts:AssumeRole`, target trust policies, trust conditions, temporary credentials, IAM Policy Simulator checks, resource policies, SCP/boundary considerations, and a realistic CI-to-production AccessDenied failure.
- [DAY 10 - S3 Security, Encryption, Bucket Policies and Access Troubleshooting](DAY%2010%20-%20S3%20Security%20Encryption%20Bucket%20Policies%20and%20Access%20Troubleshooting.md) — S3 authorization layers, bucket vs object ARNs, bucket policies and explicit denies, Block Public Access, object ownership, SSE-S3/SSE-KMS/DSSE-KMS/SSE-C, KMS authorization, AWS CLI testing, policy simulation, encryption enforcement, VPC endpoint conditions, and realistic GetObject/PutObject AccessDenied troubleshooting.
- [DAY 11 - RDS HA, Backups, Multi-AZ and Connection Troubleshooting](DAY%2011%20-%20RDS%20HA%20Backups%20Multi-AZ%20and%20Connection%20Troubleshooting.md) — RDS connection path, DNS endpoint troubleshooting, database ports, Security Groups/NACLs, DB subnet groups, Multi-AZ high availability and failover, automated backups and recovery, snapshots, AWS CLI inspection, and a realistic RDS connection timeout caused by an incorrect application Security Group.
- [DAY 12 - CloudWatch, CloudTrail and Incident Investigation](DAY%2012%20-%20CloudWatch%20CloudTrail%20and%20Incident%20Investigation.md) — CloudWatch metrics and alarms, Logs Insights, CloudTrail API events, incident timelines, evidence correlation, AWS CLI investigation, and realistic RDS/ALB incident response scenarios.

## Centralized Resources

- [LABS](LABS.md)
- [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md)
