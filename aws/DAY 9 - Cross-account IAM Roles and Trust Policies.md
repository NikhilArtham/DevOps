# AWS DAY 9 — Cross-account IAM Roles and Trust Policies

## 1. What Cross-Account IAM Means

Cross-account access means a principal in one AWS account accesses a resource in another AWS account.

A common DevOps architecture is:

```text
Account A — CI / tooling
        |
        | sts:AssumeRole
        ↓
Account B — workload / production
        |
        ↓
CrossAccountDeployRole
        |
        ↓
AWS API / resource
```

The key idea is that the caller does not simply "get permission" in the target account. The caller must be allowed to assume a role, and the assumed role must have permission to perform the required API call.

Think of cross-account authorization as two gates:

```text
GATE 1 — Can the principal assume the target role?
             ↓
        Trust policy
             ↓
GATE 2 — Once assumed, can the role perform the API call?
             ↓
        Identity policy + resource policy + conditions + boundaries/SCPs/etc.
```

A role can therefore have a correct permissions policy and still be impossible to assume because its trust policy is wrong.

---

## 2. Trust Policy vs Permissions Policy

This distinction is one of the most important IAM interview concepts.

### Trust policy

The role's trust policy answers:

```text
WHO is allowed to assume this role?
```

It is a resource-based policy attached to the IAM role.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/CIRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Permissions policy

The role's permissions policy answers:

```text
WHAT can the assumed role do?
```

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::prod-artifacts/releases/*"
    }
  ]
}
```

Mental model:

```text
Trust policy
→ Can I enter the role?

Permissions policy
→ What can I do after entering the role?
```

---

## 3. Cross-Account AssumeRole Flow

Suppose:

```text
Account A: 111111111111
Account B: 222222222222

Account A role:
CIRole

Account B role:
ProdDeployRole
```

The flow is:

```text
CI in Account A
      ↓
CIRole credentials
      ↓
sts:AssumeRole
      ↓
arn:aws:iam::222222222222:role/ProdDeployRole
      ↓
Temporary credentials
      ↓
AWS API call in Account B
```

The target role's trust policy must trust the appropriate principal from Account A.

The source principal also needs permission to call:

```text
sts:AssumeRole
```

on the target role ARN, unless the applicable authorization model provides the required permission another way.

Therefore troubleshoot both sides.

---

## 4. Source Permission vs Target Trust

A common failure is:

```text
Source role:
Allow sts:AssumeRole

Target role:
Trust policy does NOT trust source role

Result:
AccessDenied
```

Another failure is the reverse:

```text
Target role:
Trusts source role

Source role:
Does NOT allow sts:AssumeRole on target role

Result:
AccessDenied
```

The practical model is:

```text
Source-side authorization
        +
Target role trust
        ↓
AssumeRole succeeds
```

Then:

```text
Temporary role credentials
        ↓
Target-role permissions
        ↓
Target API call
```

---

## 5. Why Trust the Specific Role Instead of the Whole Account?

A trust policy can trust an entire account:

```json
"Principal": {
  "AWS": "arn:aws:iam::111111111111:root"
}
```

This does not mean the root user itself is necessarily the only principal involved. In IAM trust-policy semantics, trusting an account principal can allow principals from that account that are authorized to assume the role.

For least privilege, a more specific trust relationship is often preferable:

```json
"Principal": {
  "AWS": "arn:aws:iam::111111111111:role/CIRole"
}
```

Now the trust relationship clearly identifies the intended source role.

Do not blindly use account-wide trust when only one automation role needs access.

---

## 6. Conditions in Trust Policies

Trust policies can restrict assumption further with conditions.

Example:

```json
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "example-external-id"
  }
}
```

Other condition keys can restrict the request based on the identity or request context where supported.

The mental model is:

```text
Trusted principal
       ↓
Condition checks
       ↓
AssumeRole allowed or denied
```

For external third-party access, an ExternalId can help prevent the confused-deputy problem when the integration supports this pattern.

For AWS-native account-to-account automation, prefer conditions that actually match the architecture and available request context. Do not add conditions simply because they look secure.

---

## 7. Role ARN vs Assumed-Role Session ARN

When troubleshooting, distinguish:

```text
Role ARN:
arn:aws:iam::222222222222:role/ProdDeployRole
```

from a temporary session identity such as:

```text
arn:aws:sts::222222222222:assumed-role/ProdDeployRole/ci-session
```

The session is created after `AssumeRole` succeeds.

The role ARN identifies the IAM role. The assumed-role ARN identifies a temporary session of that role.

This distinction matters when reading CloudTrail events, policies, logs, and caller identity output.

---

## 8. AWS CLI — Inspect the Current Identity

Always verify who is actually making the request:

```bash
aws sts get-caller-identity
```

Useful output fields include:

```text
Account
Arn
UserId
```

For an assumed role, the ARN normally shows the assumed-role session.

Do not assume that the shell, CI job name, or instance name is the AWS principal.

---

## 9. AWS CLI — Inspect a Role and Trust Policy

Get the role:

```bash
aws iam get-role \
  --role-name ProdDeployRole
```

The response includes the role's `AssumeRolePolicyDocument`.

You can query the trust policy directly from the response:

```bash
aws iam get-role \
  --role-name ProdDeployRole \
  --query 'Role.AssumeRolePolicyDocument'
```

List attached policies:

```bash
aws iam list-attached-role-policies \
  --role-name ProdDeployRole
```

List inline policies:

```bash
aws iam list-role-policies \
  --role-name ProdDeployRole
```

These commands separate:

```text
Who can assume the role?
        ↓
Trust policy

What can the role do?
        ↓
Permissions policies
```

---

## 10. AWS CLI — Assume the Role

From the source account, test the role directly:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/ProdDeployRole \
  --role-session-name devops-test
```

The response contains temporary credentials.

For a controlled test, export them:

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
```

Then verify:

```bash
aws sts get-caller-identity
```

Expected identity:

```text
arn:aws:sts::222222222222:assumed-role/ProdDeployRole/devops-test
```

Avoid putting credentials in scripts, source control, or chat. Prefer your normal AWS credential/session-management mechanism for real environments.

---

## 11. Testing the Permission Path

Cross-account troubleshooting should test the complete chain rather than only one policy.

Use this sequence:

```text
1. Identify source principal
2. Identify target role ARN
3. Check source permission for sts:AssumeRole
4. Check target role trust policy
5. Check trust-policy conditions
6. Assume the role
7. Verify assumed identity
8. Identify failing target API action
9. Check target-role permissions
10. Check resource policy where applicable
11. Check explicit denies / SCP / boundary / session constraints
12. Test the exact API call
```

This prevents the common mistake of saying "the role has S3 permission" while never proving that the role can actually be assumed.

---

## 12. IAM Policy Simulator / Equivalent Checks

For a target-role API permission, the IAM Policy Simulator can help evaluate a principal's effective permissions.

Example:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::222222222222:role/ProdDeployRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::prod-artifacts/releases/app.zip
```

For the source-side AssumeRole permission, test the source principal with:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111111111111:role/CIRole \
  --action-names sts:AssumeRole \
  --resource-arns arn:aws:iam::222222222222:role/ProdDeployRole
```

But remember: a simulator result is only one piece of evidence. The target role's trust policy is essential to the real cross-account AssumeRole path.

Also inspect organizational controls such as SCPs and conditions when they apply.

---

## 13. Resource Policies in Cross-Account Access

Cross-account access can involve both IAM roles and resource-based policies.

Example:

```text
Account A
  CI role
    ↓ AssumeRole
Account B
  Deploy role
    ↓ s3:GetObject
  S3 bucket
```

The target role may have:

```text
Allow s3:GetObject
```

while the S3 bucket policy may add restrictions.

Do not assume that a role permission automatically means the resource will accept the request.

Check:

```text
Identity policy
Resource policy
Conditions
Explicit denies
SCPs
Permissions boundary
Session policy
```

The exact evaluation depends on the service and authorization model.

---

## 14. Realistic Failure Scenario — Cross-Account CI Deployment

### Architecture

```text
Account A — CI

GitHub Actions / Jenkins
        ↓
CIRole
        ↓
sts:AssumeRole
        ↓
Account B — Production
        ↓
ProdDeployRole
        ↓
S3 / ECR / ECS / Lambda deployment APIs
```

Incident:

```text
CI pipeline fails:
AccessDenied when calling AssumeRole
```

### Step 1 — Identify the source principal

Run:

```bash
aws sts get-caller-identity
```

Suppose:

```text
arn:aws:iam::111111111111:role/CIRole
```

### Step 2 — Identify the target role

```text
arn:aws:iam::222222222222:role/ProdDeployRole
```

### Step 3 — Test source-side permission

Simulate:

```text
sts:AssumeRole
```

against the target role ARN.

Suppose the result is:

```text
allowed
```

### Step 4 — Inspect target trust policy

Suppose it trusts:

```json
"Principal": {
  "AWS": "arn:aws:iam::111111111111:role/OldCIRole"
}
```

But the pipeline now uses:

```text
CIRole
```

Therefore:

```text
Source permission → ALLOW
Target trust       → does not trust actual source

AssumeRole         → DENIED
```

### Step 5 — Least-privilege fix

Do not change the source role to AdministratorAccess.

Do not trust the entire organization/account if one role is sufficient.

Update the target trust relationship to trust the exact intended source role:

```json
"Principal": {
  "AWS": "arn:aws:iam::111111111111:role/CIRole"
}
```

Then test:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/ProdDeployRole \
  --role-session-name devops-test
```

Verify with:

```bash
aws sts get-caller-identity
```

Finally test the exact deployment API call using the temporary target-role credentials.

---

## 15. Second Failure — AssumeRole Works but API Call Fails

Do not stop after fixing the trust policy.

Suppose:

```text
AssumeRole → SUCCESS
s3:GetObject → AccessDenied
```

Now the trust relationship is not the problem.

Investigate the target role's effective permissions:

```text
Target role identity policy
        ↓
S3 bucket policy
        ↓
Conditions
        ↓
Explicit Deny
        ↓
SCP / boundary / session constraints
        ↓
Effective authorization
```

This is why cross-account troubleshooting has two separate authorization stages.

---

## 16. Common Cross-Account IAM Mistakes

### Mistake 1 — Confusing trust with permission

Trust policy answers who may assume the role. Permissions answer what the role can do.

### Mistake 2 — Checking only the target role

The source principal also needs the required authorization to call `sts:AssumeRole`.

### Mistake 3 — Checking only the source role

The target role must trust the correct principal.

### Mistake 4 — Trusting the entire account unnecessarily

Use a specific source role where that is the actual requirement.

### Mistake 5 — Fixing AccessDenied with AdministratorAccess

This hides the authorization problem and creates unnecessary privilege.

### Mistake 6 — Forgetting SCPs and boundaries

A role's policy can look correct while organizational or principal-level controls limit the effective permission.

### Mistake 7 — Not verifying the assumed identity

Always run:

```bash
aws sts get-caller-identity
```

after assumption.

---

## 17. Production Troubleshooting Decision Tree

```text
Cross-account API call fails
          ↓
Which identity is calling?
          ↓
aws sts get-caller-identity
          ↓
AssumeRole itself failing?
   ├── YES
   │     ↓
   │  Check source sts:AssumeRole permission
   │     ↓
   │  Check target trust policy
   │     ↓
   │  Check trust conditions
   │     ↓
   │  Check SCP / explicit deny where applicable
   │
   └── NO
         ↓
      Verify assumed-role identity
         ↓
      Identify exact API action + resource
         ↓
      Check target role permissions
         ↓
      Check resource policy
         ↓
      Check conditions / explicit deny / boundary / SCP
         ↓
      Policy Simulator / equivalent checks
         ↓
      Apply least-privilege fix
         ↓
      Retest exact API call
```

---

## 18. Interview-Ready Mental Model

```text
ACCOUNT A
Source principal
      ↓
Can source call sts:AssumeRole?
      ↓
ACCOUNT B
Does target role trust source principal?
      ↓
AssumeRole succeeds
      ↓
Temporary credentials
      ↓
What API action?
      ↓
What resource?
      ↓
Target role permissions
      ↓
Resource policy / conditions
      ↓
Explicit deny / SCP / boundary / session constraints
      ↓
Effective permission
```

One-minute answer:

> In cross-account IAM, I separate role assumption from the permissions of the assumed role. First I identify the actual source principal with `aws sts get-caller-identity`, then verify that the source is authorized for `sts:AssumeRole` on the target role and that the target role's trust policy allows that exact principal. After assuming the role, I verify the temporary identity and investigate the failing API action using the target role's identity policy, applicable resource policy, conditions, explicit denies, SCPs, boundaries and session constraints. I validate the permission path with the IAM Policy Simulator or equivalent checks and fix only the missing principal, action, resource or condition rather than granting broad access.

---

## Beginner Takeaway

You should now be able to explain:

- What cross-account IAM access means.
- Trust policy vs permissions policy.
- Source-side `sts:AssumeRole` permission vs target-side trust.
- How temporary role credentials are created.
- Role ARN vs assumed-role session ARN.
- How to inspect trust policies with AWS CLI.
- How to test AssumeRole and verify the resulting identity.
- How resource policies, conditions, SCPs and boundaries can still affect the final API call.
- How to diagnose one failing API call and apply a least-privilege fix.
