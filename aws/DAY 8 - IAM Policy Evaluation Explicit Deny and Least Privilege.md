# AWS DAY 8 — IAM Policy Evaluation, Explicit Deny and Least Privilege

## 1. What IAM Authorization Is Actually Deciding

When an AWS API call is made, IAM must answer one question:

```text
Is this principal allowed to perform this action on this resource under these conditions?
```

A useful troubleshooting model is:

```text
Principal
   ↓
Identity-based policies
   ↓
Resource-based policies
   ↓
Permissions boundaries / session policies / SCPs where applicable
   ↓
Conditions
   ↓
Explicit Deny?
   ↓
Final decision
```

The important mental model is:

```text
Explicit Deny → DENY
Explicit Allow → may allow
No applicable Allow → DENY
```

Do not think of IAM as simply "attach AdministratorAccess and try again." The goal is to identify the exact authorization layer that decides the request.

---

## 2. Principal, Action, Resource and Condition

Every IAM authorization investigation should start by identifying four things:

```text
WHO    → principal
WHAT   → action
WHERE  → resource
UNDER WHAT CONDITIONS → condition
```

Example:

```text
Principal: IAM role DevOpsDeployRole
Action:    s3:GetObject
Resource:  arn:aws:s3:::company-artifacts/prod/app.zip
Condition: aws:RequestedRegion = ap-south-1
```

If any of these are misunderstood, the policy investigation can go in the wrong direction.

---

## 3. Identity-Based Policies

Identity-based policies are attached to IAM identities such as:

```text
User
Group
Role
```

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-artifacts/prod/*"
    }
  ]
}
```

The important pieces are:

- `Effect` → Allow or Deny
- `Action` → API operation
- `Resource` → ARN being accessed
- `Condition` → optional constraints

A role can have several policies. You must evaluate the applicable statements together.

---

## 4. Resource-Based Policies

Some AWS resources can have policies attached directly to the resource.

Common examples include:

```text
S3 bucket policy
SQS queue policy
SNS topic policy
KMS key policy
Secrets Manager resource policies where supported
```

Therefore an access failure should not automatically be investigated only from the IAM role.

Think:

```text
Identity policy
       +
Resource policy
       +
Conditions
       +
Other policy controls
```

The exact evaluation behavior varies by AWS service and policy type, so verify the service-specific authorization model when troubleshooting an unusual case.

---

## 5. Explicit Deny

An explicit deny is the strongest IAM troubleshooting signal.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::company-artifacts/*"
    }
  ]
}
```

Even if another policy says:

```text
Allow s3:DeleteObject
```

the explicit deny wins.

Mental model:

```text
Allow
  ↓
Deny exists?
  ↓
YES → DENY
NO  → continue evaluation
```

A common production mistake is finding an Allow statement and stopping the investigation.

---

## 6. Where Explicit Denies Can Come From

When an API call is unexpectedly denied, look beyond the attached IAM role policy.

Potential sources include:

```text
Identity-based policy
Resource-based policy
Permissions boundary
Session policy
AWS Organizations SCP
VPC endpoint policy where applicable
KMS key policy
Service-specific controls
```

The exact combination depends on the AWS service and request path.

This is why the question:

> "Does the role have permission?"

is often too simple.

The better question is:

> "What is the complete authorization path for this principal, action, resource and request context?"

---

## 7. Permissions Boundaries

A permissions boundary limits the maximum permissions an IAM principal can receive through identity-based policies.

Conceptually:

```text
Identity policy
      ∩
Permissions boundary
      ↓
Maximum effective permissions
```

Example:

```text
Identity policy:
Allow s3:GetObject
Allow s3:PutObject
Allow ec2:DescribeInstances

Boundary:
Allow only s3:GetObject

Effective permission:
→ s3:GetObject
```

The identity policy alone can therefore look correct while the effective permission is still denied.

---

## 8. Session Policies

A session policy can further restrict permissions during a role session.

Think of the session as another authorization constraint:

```text
Role permissions
      ↓
Session policy constraints
      ↓
Effective permissions
```

This matters when troubleshooting assumed roles, federated access, or automation that passes session policies.

---

## 9. AWS Organizations SCPs

A Service Control Policy (SCP) defines the maximum available permissions for accounts or organizational units.

Important:

```text
SCP does not itself grant permissions.
```

Instead, it limits what permissions can be exercised in the account.

Conceptually:

```text
IAM Allow
   ∩
SCP Allow boundary
   ↓
Effective authorization
```

An account administrator can therefore have an identity policy that allows an action while an organization-level SCP prevents it.

---

## 10. Conditions Matter

A policy can allow an action only under specific conditions.

Example:

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": "ap-south-1"
  }
}
```

The action may succeed in `ap-south-1` but fail in another region.

Other common condition concepts include:

```text
aws:PrincipalArn
aws:SourceVpce
aws:SourceIp
aws:SecureTransport
aws:RequestedRegion
aws:MultiFactorAuthPresent
```

Do not remove a condition just because it causes a denial. First determine why the condition exists.

---

## 11. IAM Policy Evaluation Mental Model

For a DevOps troubleshooting interview, use this simplified model:

```text
1. Identify principal
2. Identify API action
3. Identify resource
4. Collect applicable identity policies
5. Collect applicable resource policies
6. Check conditions
7. Check permissions boundary/session policy/SCP where applicable
8. Search for explicit Deny
9. Determine effective permission
10. Apply least-privilege fix
```

The most important rule:

```text
Explicit Deny overrides Allow.
```

And:

```text
No applicable Allow = Deny.
```

---

## 12. AWS CLI Investigation Commands

### Identify the caller

For temporary credentials or an assumed role:

```bash
aws sts get-caller-identity
```

This is one of the first commands to run because you may be troubleshooting the wrong identity.

### Inspect a role

```bash
aws iam get-role --role-name <role-name>
```

### List attached managed policies

```bash
aws iam list-attached-role-policies --role-name <role-name>
```

### List inline policies

```bash
aws iam list-role-policies --role-name <role-name>
```

### Inspect a policy version

```bash
aws iam get-policy-version \
  --policy-arn <policy-arn> \
  --version-id <version-id>
```

### Check a role's permissions boundary

```bash
aws iam get-role \
  --role-name <role-name> \
  --query 'Role.PermissionsBoundary'
```

### Inspect an S3 bucket policy

```bash
aws s3api get-bucket-policy --bucket <bucket-name>
```

The command components matter:

```text
aws       → AWS CLI
s3api     → S3 API command group
get-bucket-policy → retrieve resource policy
--bucket  → target bucket
```

---

## 13. Policy Simulator

The IAM Policy Simulator can test whether a principal is allowed to perform specific actions against resources.

A useful CLI command is:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/DevOpsRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::company-artifacts/prod/app.zip
```

Important arguments:

```text
--policy-source-arn → principal/policy source being evaluated
--action-names      → API action to test
--resource-arns     → resource involved in the request
```

For a realistic investigation, add the request context and other relevant simulation parameters where supported rather than testing only the action name.

The simulator is evidence, not magic. If the actual request includes service-specific behavior or policies outside the simulator's modeled context, continue the investigation.

---

## 14. Realistic Failure Scenario — S3 GetObject AccessDenied

### Architecture

```text
GitHub Actions / CI
       ↓
AssumeRole
       ↓
DevOpsDeployRole
       ↓
s3:GetObject
       ↓
S3 bucket/object
```

Incident:

```text
aws s3 cp s3://company-artifacts/prod/app.zip .

AccessDenied
```

### Step 1 — Identify the actual principal

```bash
aws sts get-caller-identity
```

Suppose the caller is:

```text
arn:aws:iam::123456789012:role/DevOpsDeployRole
```

Do not assume the local username or CI service name is the effective principal.

### Step 2 — Identify the exact API action

The failing operation requires an object read permission:

```text
s3:GetObject
```

### Step 3 — Identify the exact resource

```text
arn:aws:s3:::company-artifacts/prod/app.zip
```

### Step 4 — Inspect identity policy

Suppose the role allows:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::company-artifacts/prod/*"
}
```

At first glance this looks correct.

### Step 5 — Inspect resource policy

Suppose the bucket policy contains an explicit deny:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::company-artifacts/*",
  "Condition": {
    "StringNotEquals": {
      "aws:SourceVpce": "vpce-0123456789abcdef0"
    }
  }
}
```

The request is coming through a different path, so the condition matches the deny.

Result:

```text
Identity policy → ALLOW
Bucket policy   → EXPLICIT DENY

Effective result → DENY
```

### Step 6 — Prove it

Use the policy simulator/equivalent IAM checks and inspect the actual request path.

Then validate the S3 access from the intended network path.

### Step 7 — Least-privilege fix

Do **not** solve this with:

```text
AdministratorAccess
```

Instead determine the intended architecture.

If the bucket is deliberately restricted to a VPC endpoint, make the workload use the approved endpoint path and keep the restrictive policy.

If the policy condition is obsolete, change only the specific condition or statement through the approved review process.

The goal is:

```text
Minimum principal
+ Minimum action
+ Minimum resource
+ Required conditions
= Least privilege
```

---

## 15. Least Privilege

Least privilege means granting only the permissions required for a workload to perform its intended function.

Bad:

```json
"Action": "*",
"Resource": "*"
```

Better:

```json
"Action": [
  "s3:GetObject"
],
"Resource": "arn:aws:s3:::company-artifacts/prod/*"
```

Even better where architecture allows:

```text
Specific role
Specific API actions
Specific resources
Specific conditions
```

Least privilege is not simply "make the policy smaller." It is about matching permissions to the actual operational requirement.

---

## 16. Common IAM Troubleshooting Mistakes

### Mistake 1 — Looking only for Allow

An Allow does not prove access if an explicit Deny applies.

### Mistake 2 — Troubleshooting the wrong principal

Always start with:

```bash
aws sts get-caller-identity
```

### Mistake 3 — Ignoring resource policies

S3, SQS, SNS and other services can have resource-based authorization controls.

### Mistake 4 — Ignoring conditions

A policy may allow an action only from a specific VPC endpoint, region, IP range, TLS state, or other context.

### Mistake 5 — Adding AdministratorAccess

This destroys the evidence and creates unnecessary privilege.

### Mistake 6 — Confusing IAM permission with application permission

IAM can allow the API call while the application still fails because of encryption, object ownership, network access, or application-level authorization.

---

## 17. Production Troubleshooting Decision Tree

```text
API call fails with AccessDenied
          ↓
aws sts get-caller-identity
          ↓
Correct principal?
   ├── NO → fix credentials/role assumption
   └── YES
          ↓
Identify exact API action + resource
          ↓
Inspect identity policy
          ↓
Inspect resource policy where applicable
          ↓
Check conditions
          ↓
Check boundary/session policy/SCP where applicable
          ↓
Search for explicit Deny
          ↓
Policy Simulator / equivalent evidence
          ↓
Least-privilege correction
          ↓
Retest exact API call
```

---

## 18. Interview-Ready Mental Model

```text
WHO?
 ↓
WHAT ACTION?
 ↓
WHICH RESOURCE?
 ↓
WHICH CONDITIONS?
 ↓
WHICH POLICIES APPLY?
 ↓
EXPLICIT DENY?
 ↓
EFFECTIVE PERMISSION
 ↓
LEAST-PRIVILEGE FIX
```

One-minute answer:

> When an AWS API call is denied, I first identify the actual principal with `aws sts get-caller-identity`, then identify the exact API action and resource. I inspect identity-based and applicable resource-based policies, conditions, permissions boundaries, session policies and SCPs where relevant. I specifically search for an explicit Deny because it overrides an Allow. I validate the authorization path with the IAM Policy Simulator or equivalent AWS checks. Finally, I fix the smallest permission boundary needed — specific principal, action, resource and conditions — instead of granting broad permissions such as AdministratorAccess.

---

## Beginner Takeaway

You should now be able to explain:

- Identity-based vs resource-based policies.
- Principal, action, resource and condition.
- Explicit Deny vs Allow.
- Why an Allow statement does not always mean access is granted.
- Permissions boundaries, session policies and SCPs.
- How to identify the real caller with STS.
- How to use the IAM Policy Simulator.
- How to troubleshoot an `AccessDenied` API call systematically.
- How to apply least privilege without using AdministratorAccess as a shortcut.
