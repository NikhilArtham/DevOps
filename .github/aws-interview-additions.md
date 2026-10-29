
---

## DAY 6 — EC2 Boot, User-Data, Metadata and System Troubleshooting

### 1. What is the difference between EC2 instance state and status checks?
<details><summary>Answer</summary>Instance state describes whether the instance is running/stopped. Status checks test underlying system and instance reachability/health. A running instance can still have an application failure.</details>

### 2. What is user-data?
<details><summary>Answer</summary>User-data is bootstrap input used to initialize an EC2 instance, commonly for package installation and service configuration.</details>

### 3. How do you troubleshoot cloud-init failure?
<details><summary>Answer</summary>Check `cloud-init status --long`, `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log`, then verify the intended service and listener.</details>

### 4. What is IMDSv2?
<details><summary>Answer</summary>It is the token-based EC2 Instance Metadata Service. A client obtains a session token before retrieving metadata.</details>

### 5. Why can EC2 be healthy while an ALB target is unhealthy?
<details><summary>Answer</summary>The instance can be running while bootstrap failed, the application is not listening, the health path/port is wrong, or network/security controls block the health check.</details>

### 6. When would you use SSM instead of SSH?
<details><summary>Answer</summary>For managed, IAM-authorized access without exposing inbound SSH, especially for private instances.</details>

### 7. What does `ss -lntp` prove?
<details><summary>Answer</summary>It shows listening TCP sockets and associated processes where permitted, helping prove whether the application is actually listening.</details>

### 8. Why can localhost work while a remote health check fails?
<details><summary>Answer</summary>The application may be bound only to `127.0.0.1`, so local requests work but remote traffic cannot reach it.</details>

### 9. What should you check before changing a Security Group during an EC2 boot incident?
<details><summary>Answer</summary>Check status checks, cloud-init/systemd logs, listening ports and local application health first. Prove the network layer is actually failing.</details>

### 10. Give the EC2 troubleshooting sequence.
<details><summary>Answer</summary>Instance state → status checks → network path → access method → bootstrap/system logs → listening port → application health.</details>

---

## DAY 7 — Systems Manager: Session Manager, Run Command and Patch

### 1. What is Systems Manager?
<details><summary>Answer</summary>AWS Systems Manager provides centralized operational capabilities for managed nodes, including Session Manager, Run Command and patching.</details>

### 2. Does Session Manager require inbound SSH?
<details><summary>Answer</summary>No. It can provide access without inbound port 22 when the agent, IAM and network prerequisites are satisfied.</details>

### 3. What are the main Session Manager prerequisites?
<details><summary>Answer</summary>A healthy SSM Agent, suitable IAM permissions and network connectivity to required Systems Manager endpoints.</details>

### 4. What is Run Command?
<details><summary>Answer</summary>It executes commands or SSM documents remotely on managed nodes.</details>

### 5. Why might Run Command remain Pending?
<details><summary>Answer</summary>The node may not be registered/online, the agent may be unhealthy, or the instance may lack connectivity to Systems Manager services.</details>

### 6. How do you troubleshoot a node missing from SSM?
<details><summary>Answer</summary>Check `describe-instance-information`, the SSM Agent, IAM permissions, endpoint/network connectivity and account/region.</details>

### 7. What is Patch Manager used for?
<details><summary>Answer</summary>It manages patch operations and assesses patch compliance across managed nodes.</details>

### 8. Why is SSM useful for private subnets?
<details><summary>Answer</summary>It can provide operational access without public IPs or exposed SSH, while still requiring private connectivity to Systems Manager.</details>

### 9. What should you prove before changing Security Groups for SSM?
<details><summary>Answer</summary>Prove managed-node registration, agent health, IAM and endpoint/network connectivity first.</details>

### 10. Explain Session Manager troubleshooting in one minute.
<details><summary>Answer</summary>Verify managed-node registration → agent → IAM → endpoint/network connectivity → session/target configuration.</details>

---

## DAY 8 — IAM Policy Evaluation, Explicit Deny and Least Privilege

### 1. What is the IAM evaluation model?
<details><summary>Answer</summary>Applicable identity/resource policies and constraints are evaluated for the request. An applicable explicit Deny overrides an Allow.</details>

### 2. Identity policy vs resource policy?
<details><summary>Answer</summary>Identity policies attach to principals; resource policies attach to resources such as S3 buckets.</details>

### 3. What is an explicit Deny?
<details><summary>Answer</summary>A matching Deny overrides an Allow and is commonly used for security guardrails.</details>

### 4. What is a permissions boundary?
<details><summary>Answer</summary>It defines the maximum permissions an identity-based policy can grant to a principal.</details>

### 5. What is an SCP?
<details><summary>Answer</summary>An SCP limits maximum available permissions for accounts/OUs; it does not grant permissions itself.</details>

### 6. Why start with `aws sts get-caller-identity`?
<details><summary>Answer</summary>You need to know which principal actually made the request before evaluating its policies.</details>

### 7. What does the Policy Simulator help prove?
<details><summary>Answer</summary>It evaluates whether a principal is allowed or denied for specified actions/resources, especially identity-policy behavior.</details>

### 8. Why not use AdministratorAccess as the fix?
<details><summary>Answer</summary>It hides the root cause, violates least privilege and increases production risk.</details>

### 9. Can an Allow overcome an explicit Deny?
<details><summary>Answer</summary>No. An applicable explicit Deny wins.</details>

### 10. Give an IAM AccessDenied sequence.
<details><summary>Answer</summary>Caller → action → resource → identity policy → resource policy → boundary/session policy → SCP → conditions → explicit Deny → retest.</details>

---

## DAY 9 — Cross-account IAM Roles and Trust Policies

### 1. What are the two main AssumeRole gates?
<details><summary>Answer</summary>The source principal needs `sts:AssumeRole` permission and the target role trust policy must trust that principal under the required conditions.</details>

### 2. Trust policy vs permissions policy?
<details><summary>Answer</summary>Trust controls who may assume the role; permissions control what the assumed role may do.</details>

### 3. Does a trust policy grant S3 access?
<details><summary>Answer</summary>No. It allows assumption of the role. Target permissions determine the role session's API access.</details>

### 4. How do you verify the assumed identity?
<details><summary>Answer</summary>Use `aws sts get-caller-identity` with the temporary credentials.</details>

### 5. Why trust an exact CI role instead of the entire account?
<details><summary>Answer</summary>It narrows the principals allowed to assume the role and follows least privilege.</details>

### 6. What is ExternalId for?
<details><summary>Answer</summary>It can be used in trust conditions for third-party access to reduce confused-deputy risk.</details>

### 7. AssumeRole succeeds but S3 fails. What does that indicate?
<details><summary>Answer</summary>The basic trust/AssumeRole path worked; investigate target-role permissions, resource policies, KMS and organization/session constraints.</details>

### 8. What is a common cross-account mistake?
<details><summary>Answer</summary>Changing target permissions when trust is failing, or changing trust after AssumeRole already succeeds.</details>

### 9. What should be tested separately?
<details><summary>Answer</summary>Test source `sts:AssumeRole` authorization separately from the assumed role's target API authorization.</details>

### 10. Explain cross-account access in one minute.
<details><summary>Answer</summary>Source permission plus target trust must permit AssumeRole; the resulting role session then needs target API permissions, subject to resource policies, SCPs, boundaries, session policies, conditions and explicit denies.</details>

---

## DAY 10 — S3 Security, Encryption, Bucket Policies and Access Troubleshooting

### 1. Bucket ARN vs object ARN?
<details><summary>Answer</summary>The bucket ARN is `arn:aws:s3:::bucket-name`; object resources commonly use `arn:aws:s3:::bucket-name/*`. `ListBucket` and `GetObject` therefore usually use different resource ARNs.</details>

### 2. What is S3 Block Public Access?
<details><summary>Answer</summary>It provides controls designed to prevent public access configurations at account/bucket scope.</details>

### 3. What is a bucket policy?
<details><summary>Answer</summary>A resource-based policy defining which principals can access the bucket/objects and under what conditions.</details>

### 4. Can an Allow override an explicit Deny?
<details><summary>Answer</summary>No. An applicable explicit Deny wins.</details>

### 5. SSE-S3 vs SSE-KMS?
<details><summary>Answer</summary>Both provide server-side encryption, but SSE-KMS uses KMS keys and introduces separate KMS authorization/key-policy considerations.</details>

### 6. Why can GetObject fail when s3:GetObject is allowed?
<details><summary>Answer</summary>Bucket-policy Deny, Block Public Access, conditions, SCP/boundary/session restrictions or KMS authorization can still block the request.</details>

### 7. What should you inspect for SSE-KMS AccessDenied?
<details><summary>Answer</summary>Inspect key state, key policy, required KMS permissions and explicit deny/conditions. S3 authorization alone is not sufficient.</details>

### 8. How do you troubleshoot S3 AccessDenied?
<details><summary>Answer</summary>Identify caller/action/resource, then inspect IAM, bucket policy, Block Public Access, ownership/ACLs, encryption/KMS, endpoint policy, SCP/boundary, conditions and explicit denies. Retest the exact operation.</details>

### 9. What does `aws:SecureTransport` enforce?
<details><summary>Answer</summary>A bucket policy can deny requests when secure transport is false, enforcing HTTPS for the access path.</details>

### 10. How can `aws:SourceVpce` affect access?
<details><summary>Answer</summary>A bucket policy can restrict access to a specific VPC endpoint, so the same principal can succeed through the approved endpoint and fail through another path.</details>

---

## DAY 11 — RDS HA, Backups, Multi-AZ and Connection Troubleshooting

### 1. What is the RDS connection path?
<details><summary>Answer</summary>Client → RDS DNS endpoint → DNS resolution → network path → RDS Security Group → database port → RDS state → TLS/authentication → database.</details>

### 2. What is Multi-AZ in RDS?
<details><summary>Answer</summary>Multi-AZ provides a standby deployment in another Availability Zone for high availability and managed failover. It is primarily an availability feature, not standard read scaling.</details>

### 3. What are automated RDS backups used for?
<details><summary>Answer</summary>They support recovery objectives such as point-in-time recovery within the configured retention period.</details>

### 4. Does Multi-AZ replace backups?
<details><summary>Answer</summary>No. Multi-AZ addresses availability/failover; backups address data recovery and point-in-time recovery.</details>

### 5. What should you check first when an RDS client cannot connect?
<details><summary>Answer</summary>Check the RDS endpoint/DNS, then TCP reachability to the actual database port, Security Groups/NACLs/network path, RDS status and finally authentication/application behavior.</details>

### 6. DNS resolves but TCP times out. What layer is most likely?
<details><summary>Answer</summary>The network or security layer: routing, NACL, Security Group or other network reachability. It is not primarily a DNS failure.</details>

### 7. TCP connects but the database rejects the client. What does that prove?
<details><summary>Answer</summary>The DNS and basic TCP path worked. Investigate database authentication, authorization, TLS, credentials or application configuration.</details>

### 8. How do you inspect Multi-AZ and backup settings?
<details><summary>Answer</summary>Use `aws rds describe-db-instances --db-instance-identifier <id>` and inspect `MultiAZ`, `BackupRetentionPeriod`, `PreferredBackupWindow` and related fields.</details>

### 9. What is an RDS DB subnet group?
<details><summary>Answer</summary>It defines the subnets in which RDS can place database network interfaces. A resilient design uses appropriate subnets across Availability Zones.</details>

### 10. Why should applications use the RDS endpoint instead of a hard-coded IP?
<details><summary>Answer</summary>The managed endpoint abstracts the underlying database infrastructure and supports managed failover behavior. A hard-coded IP can become invalid after infrastructure changes.</details>

### 11. Can a running/available RDS instance still have connection failures?
<details><summary>Answer</summary>Yes. Available state does not prove DNS, routing, Security Group, port, TLS or authentication are correct.</details>

### 12. Multi-AZ vs read replica?
<details><summary>Answer</summary>Multi-AZ primarily provides high availability/failover. Read replicas primarily support read scaling and replication-based workloads.</details>

### 13. What should you inspect if an RDS Security Group looks correct?
<details><summary>Answer</summary>Verify the actual client source, destination port, attached RDS SG, subnet/NACL path, DNS result and endpoint. A rule for the wrong source or port is still wrong.</details>

### 14. Why is `nc -vz` useful for RDS?
<details><summary>Answer</summary>It tests TCP reachability without requiring database authentication, separating network/port failures from database-login failures.</details>

### 15. Does a backup fix a blocked Security Group?
<details><summary>Answer</summary>No. Backups are a recovery mechanism, not a network-access mechanism.</details>

### 16. Give an RDS troubleshooting matrix.
<details><summary>Answer</summary>DNS failure → name resolution; DNS works + TCP timeout → route/SG/NACL/network; TCP connects + DB rejects → database/authentication/TLS; RDS state unhealthy → AWS database/service layer.</details>

### 17. Why does Multi-AZ matter during an AZ failure?
<details><summary>Answer</summary>The standby in another AZ supports managed failover so the database can remain available through an eligible AZ-level failure, subject to failover behavior and application reconnection.</details>

### 18. What should an application do after RDS failover?
<details><summary>Answer</summary>Reconnect through the RDS endpoint and handle transient connection failures with appropriate retry/backoff. Do not depend on a fixed IP.</details>

### 19. What is a common RDS troubleshooting mistake?
<details><summary>Answer</summary>Changing multiple layers at once. Prove DNS, TCP, security controls, RDS state and authentication in order so the first failing layer is clear.</details>

### 20. Explain RDS troubleshooting in one minute.
<details><summary>Answer</summary>Start with endpoint/DNS, test the database port with `nc`, inspect route/NACL/SG, verify RDS state and then investigate authentication/TLS/application behavior. Keep HA and recovery separate: Multi-AZ improves availability/failover while backups support restore/PITR. Fix the first proven failing layer and retest.</details>
