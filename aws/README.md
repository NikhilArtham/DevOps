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

## Centralized Resources

- [LABS](LABS.md)
- [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md)
