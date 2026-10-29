
---

## DAY 6 — EC2 Boot, User-Data, Metadata and System Troubleshooting

### Lab 1 — EC2 State and Status Checks
```bash
aws ec2 describe-instances --instance-ids <instance-id>
aws ec2 describe-instance-status --instance-ids <instance-id> --include-all-instances
```
Record instance state, system status and instance status.

### Lab 2 — User-Data and cloud-init
```bash
cloud-init status --long
sudo tail -n 100 /var/log/cloud-init.log
sudo tail -n 100 /var/log/cloud-init-output.log
```
Find the first bootstrap failure and determine whether it prevented the application from starting.

### Lab 3 — IMDSv2
```bash
TOKEN=$(curl -sS -X PUT 'http://169.254.169.254/latest/api/token' -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')
curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```
Explain the token flow and inspect metadata options.

### Lab 4 — SSH and Application Port Troubleshooting
```bash
sudo systemctl status sshd
ss -lntp
curl -v http://127.0.0.1:<port>/health
curl -v http://<private-ip>:<port>/health
```
Separate access, listener binding and application failures.

### Lab 5 — SSM Access
```bash
aws ssm describe-instance-information
aws ssm start-session --target <instance-id>
```
Identify IAM, SSM Agent and network prerequisites.

### DAY 6 Capstone — EC2 Boot Failure
EC2 is `running`, but an ALB target is unhealthy. Use status checks, SSM, cloud-init, systemd, `ss` and `curl` to prove whether the failure is AWS networking or instance bootstrap/application startup.

---

## DAY 7 — Systems Manager: Session Manager, Run Command and Patch

### Lab 1 — Managed Node Prerequisites
```bash
aws ssm describe-instance-information
sudo systemctl status amazon-ssm-agent
```

### Lab 2 — Session Manager
```bash
aws ssm start-session --target <instance-id>
```
Explain why inbound SSH is not required.

### Lab 3 — Run Command
```bash
aws ssm send-command --instance-ids <instance-id> --document-name 'AWS-RunShellScript' --parameters 'commands=["hostname","uptime","df -h"]'
```
Inspect the result and required IAM permission.

### Lab 4 — Diagnose SSM Failure
Investigate missing/offline managed nodes, Pending commands, agent failures, IAM failures and endpoint/network failures.

### Lab 5 — Patch Compliance
Inspect missing, installed and failed patches and explain compliance state.

### DAY 7 Capstone — SSM Access Incident
EC2 is running but Session Manager cannot connect. Prove which prerequisite fails: registration, agent, IAM, endpoint/network connectivity or target state.

---

## DAY 8 — IAM Policy Evaluation, Explicit Deny and Least Privilege

### Lab 1 — Identify Caller
```bash
aws sts get-caller-identity
```

### Lab 2 — Inspect Identity Policies
```bash
aws iam list-attached-role-policies --role-name <role-name>
aws iam list-role-policies --role-name <role-name>
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version-id>
```

### Lab 3 — Inspect Resource Policy
```bash
aws s3api get-bucket-policy --bucket <bucket-name>
```

### Lab 4 — Policy Simulator
```bash
aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*
```

### Lab 5 — Explicit Deny
Create a controlled S3 case where identity policy allows an action but a resource-policy condition denies it. Prove that the explicit Deny wins.

### Lab 6 — Least-Privilege Cleanup
Reduce `s3:*` on `*` to the minimum required action/resource set and retest.

### DAY 8 Capstone — AccessDenied Decision Tree
Record caller, action, resource, identity policy, resource policy, boundary, session policy, SCP, conditions and explicit Deny. Do not use AdministratorAccess as the fix.

---

## DAY 9 — Cross-account IAM Roles and Trust Policies

### Lab 1 — Two-Gate AssumeRole Path
Document source principal → `sts:AssumeRole` permission → target trust policy → temporary session → target API permission.

### Lab 2 — Inspect Trust Policy
```bash
aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'
```

### Lab 3 — Test AssumeRole
```bash
aws sts assume-role --role-arn arn:aws:iam::<target-account>:role/<role-name> --role-session-name devops-lab
aws sts get-caller-identity
```

### Lab 4 — Separate Trust from Target Permissions
If AssumeRole succeeds but the target API fails, inspect target-role permissions, resource policies, KMS, SCPs, boundaries and conditions.

### Lab 5 — Least-Privilege Trust
Compare trusting an entire source account with trusting the exact CI role.

### DAY 9 Capstone — CI-to-Production AccessDenied
Trace Account A CI role → Account B production role → S3 API. Identify the failing gate and make only the required change.

---

## DAY 10 — S3 Security, Encryption, Bucket Policies and Access Troubleshooting

### Lab 1 — Identify Caller and Operation
```bash
aws sts get-caller-identity
aws s3api head-object --bucket <bucket> --key <key>
aws s3api get-object --bucket <bucket> --key <key> /tmp/object
```

### Lab 2 — Bucket Policy and Public Access
```bash
aws s3api get-bucket-policy --bucket <bucket>
aws s3api get-public-access-block --bucket <bucket>
aws s3api get-bucket-ownership-controls --bucket <bucket>
```

### Lab 3 — Encryption and KMS
```bash
aws s3api get-bucket-encryption --bucket <bucket>
```
If SSE-KMS is used, trace S3 authorization separately from KMS authorization.

### Lab 4 — Secure Transport and VPC Endpoint Conditions
Investigate `aws:SecureTransport` and `aws:SourceVpce` conditions and prove how network path can affect authorization.

### Lab 5 — PutObject Encryption Enforcement
Test an upload against a controlled policy requiring encryption and identify the exact Deny when the condition is missing.

### Lab 6 — S3 Access Matrix
Classify each AccessDenied case as IAM, bucket policy, Block Public Access, Object Ownership/ACL, S3 encryption, KMS, endpoint policy, SCP/boundary or condition.

### DAY 10 Capstone — SSE-KMS GetObject Failure
Trace caller → `s3:GetObject` → bucket policy → public-access controls → ownership/ACL → KMS → explicit denies/conditions. Apply the least-privilege fix and retest.

---

## DAY 11 — RDS HA, Backups, Multi-AZ and Connection Troubleshooting

### Lab 1 — Map the RDS Connection Path
```bash
aws rds describe-db-instances --db-instance-identifier <db-id> --query 'DBInstances[].{Endpoint:Endpoint.Address,Port:Endpoint.Port,Status:DBInstanceStatus,AZ:AvailabilityZone,MultiAZ:MultiAZ,Engine:Engine}'
```
Trace:
```text
Application → RDS DNS endpoint → DNS → route/network → RDS SG → DB port → RDS state → authentication
```
Use `dig`/`nslookup` and `nc -vz <endpoint> <port>`.

### Lab 2 — Inspect HA and Backup Settings
```bash
aws rds describe-db-instances --db-instance-identifier <db-id> --query 'DBInstances[].{MultiAZ:MultiAZ,BackupRetention:BackupRetentionPeriod,BackupWindow:PreferredBackupWindow,Status:DBInstanceStatus}'
```
Explain Multi-AZ/failover versus backups/PITR.

### Lab 3 — Diagnose RDS Security Group Failure
Verify the actual client source and RDS SG inbound rule for the database port. Test:
```bash
nc -vz <rds-endpoint> 5432
nc -vz <rds-endpoint> 3306
```
Use the actual engine port.

### Lab 4 — DNS vs TCP vs Database State Matrix
```text
DNS fails → name-resolution layer
DNS works + TCP timeout → route/SG/NACL/network layer
TCP connects + DB rejects → authentication/database/TLS layer
RDS unavailable → database/service state
```
Prove each result with a command or AWS observation.

### Lab 5 — Inspect DB Subnet Group
```bash
aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group>
```
Map the VPC and Availability Zones.

### Lab 6 — Failover and Endpoint Exercise
In non-production, inspect Multi-AZ and document expected failover. Verify applications use the RDS endpoint, not a hard-coded IP. Do not trigger production failover as a learning exercise.

### Lab 7 — Backup and Restore Investigation
Inspect automated-backup settings and snapshots. Explain automated backups/PITR versus manual snapshots and record the recovery workflow.

### DAY 11 Capstone — RDS Connection Incident
Scenario:
```text
Application → RDS endpoint :5432
Client reports connection timeout
RDS is running
```
Trace endpoint DNS → route/network → RDS SG → port → RDS state → HA configuration → authentication/application. Identify the exact failing layer and explain why changing backups or Multi-AZ would not fix a pure SG/port failure.
