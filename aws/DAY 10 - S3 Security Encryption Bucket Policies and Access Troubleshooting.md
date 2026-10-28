# DAY 10 — S3 Security, Encryption, Bucket Policies and Access Troubleshooting

## 1. What you are learning

S3 access failures are rarely solved by looking at only one policy.

For an S3 request, trace the full authorization and security path:

```text
Caller
  ↓
Identity / IAM policy
  ↓
S3 bucket policy
  ↓
S3 Block Public Access
  ↓
S3 object ownership / ACL considerations
  ↓
Encryption / KMS authorization
  ↓
S3 operation
```

The key DevOps skill is to identify **which layer denied the request** instead of randomly changing permissions.

---

## 2. S3 security mental model

Think about an S3 request as four questions:

1. **Who is calling?**
2. **What S3 action are they trying to perform?**
3. **Which policies allow or deny that action?**
4. **Can the object be encrypted/decrypted with the required key?**

A request can have an IAM `Allow` and still fail because another security control produces an explicit deny or because KMS authorization is missing.

```text
Principal
   |
   +--> IAM identity policy
   |
   +--> Bucket policy
   |
   +--> Block Public Access
   |
   +--> SCP / permissions boundary / session policy
   |
   +--> KMS key policy + IAM permissions
   |
   v
S3 API request
```

---

## 3. IAM policy vs bucket policy

### IAM identity policy

Attached to a user, group, or role.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::company-data-prod/reports/*"
    }
  ]
}
```

This says the identity may read objects under `reports/`.

### Bucket policy

Attached directly to the S3 bucket.

It is especially useful for:

- cross-account access
- restricting access by VPC endpoint
- requiring TLS
- restricting principals
- enforcing security conditions

Example: deny non-TLS requests.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-data-prod",
        "arn:aws:s3:::company-data-prod/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

This is a security **deny guardrail**. An IAM `Allow` does not override it.

---

## 4. Resource ARN matters

Bucket-level and object-level actions use different resource ARNs.

Bucket:

```text
arn:aws:s3:::company-data-prod
```

Objects:

```text
arn:aws:s3:::company-data-prod/*
```

For example:

- `s3:ListBucket` → bucket ARN
- `s3:GetObject` → object ARN
- `s3:PutObject` → object ARN
- `s3:DeleteObject` → object ARN

A common mistake is granting `s3:GetObject` on only the bucket ARN.

```json
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::company-data-prod"
```

That does **not** correctly target objects.

Use:

```json
"Resource": "arn:aws:s3:::company-data-prod/*"
```

---

## 5. S3 Block Public Access

S3 Block Public Access provides account/bucket-level controls to prevent public access configurations.

The four important settings are:

- `BlockPublicAcls`
- `IgnorePublicAcls`
- `BlockPublicPolicy`
- `RestrictPublicBuckets`

For production data buckets, the normal secure baseline is to keep public access blocked unless there is a documented reason for public content.

Check it with:

```bash
aws s3api get-public-access-block \
  --bucket company-data-prod
```

Check bucket policy:

```bash
aws s3api get-bucket-policy \
  --bucket company-data-prod
```

Check ACL where relevant:

```bash
aws s3api get-bucket-acl \
  --bucket company-data-prod
```

### Important distinction

An IAM policy allowing a user to access an object is not the same thing as making the object publicly accessible.

Do not solve a private-access problem by turning off Block Public Access.

---

## 6. S3 encryption

S3 supports server-side encryption options including:

- **SSE-S3** — S3-managed encryption keys
- **SSE-KMS** — AWS KMS keys
- **DSSE-KMS** — dual-layer server-side encryption with KMS
- **SSE-C** — customer-provided encryption keys

For normal enterprise workloads, SSE-S3 or SSE-KMS are common choices depending on compliance and key-management requirements.

Check bucket encryption:

```bash
aws s3api get-bucket-encryption \
  --bucket company-data-prod
```

Set a default encryption configuration using SSE-KMS:

```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:ap-south-1:111122223333:key/KEY-ID"
      },
      "BucketKeyEnabled": true
    }
  ]
}
```

### Important

S3 encryption and S3 authorization are different controls.

A caller can be authorized by S3 but still fail because the caller lacks permission to use the KMS key.

---

## 7. SSE-KMS access path

For a KMS-encrypted object, think:

```text
Caller
  ↓
S3 authorization
  ↓
S3 object
  ↓
KMS authorization
  ↓
Decrypt / encrypt
```

For downloads, a role may need permission such as:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject"
  ],
  "Resource": "arn:aws:s3:::company-data-prod/reports/*"
}
```

And, when required by the encryption setup, KMS permissions such as:

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:ap-south-1:111122223333:key/KEY-ID"
}
```

The KMS key policy must also permit the intended use. KMS authorization is therefore another layer to inspect.

---

## 8. Test the actual failing operation

Do not test only whether the bucket is reachable.

Test the exact operation.

### Identify the caller

```bash
aws sts get-caller-identity
```

### Test object read

```bash
aws s3api head-object \
  --bucket company-data-prod \
  --key reports/monthly.csv
```

Or:

```bash
aws s3 cp \
  s3://company-data-prod/reports/monthly.csv \
  ./monthly.csv
```

### Test object write

```bash
aws s3 cp \
  ./test.txt \
  s3://company-data-prod/test.txt
```

### Test bucket listing

```bash
aws s3api list-objects-v2 \
  --bucket company-data-prod \
  --max-items 5
```

These operations are not interchangeable.

A role may have `GetObject` but not `ListBucket`.

Therefore:

```text
aws s3 ls s3://bucket
```

failing does not automatically mean:

```text
aws s3 cp s3://bucket/file .
```

must fail.

---

## 9. Inspect the relevant policies

### Bucket policy

```bash
aws s3api get-bucket-policy \
  --bucket company-data-prod
```

### Public access block

```bash
aws s3api get-public-access-block \
  --bucket company-data-prod
```

### Default encryption

```bash
aws s3api get-bucket-encryption \
  --bucket company-data-prod
```

### Bucket ownership controls

```bash
aws s3api get-bucket-ownership-controls \
  --bucket company-data-prod
```

### Caller identity

```bash
aws sts get-caller-identity
```

### Role policies

```bash
aws iam list-attached-role-policies \
  --role-name DevOpsS3ReadRole

aws iam list-role-policies \
  --role-name DevOpsS3ReadRole
```

---

## 10. Policy Simulator

When the problem is authorization, test the exact principal/action/resource combination.

Example:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:role/DevOpsS3ReadRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::company-data-prod/reports/monthly.csv
```

For listing:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:role/DevOpsS3ReadRole \
  --action-names s3:ListBucket \
  --resource-arns arn:aws:s3:::company-data-prod
```

### Important limitation

The policy simulator is excellent for understanding identity-based authorization, but it should not be treated as a universal replacement for testing the real request.

S3 bucket policies, SCPs, permissions boundaries, KMS policies, network conditions, and request context may all matter.

Always correlate simulator output with the actual AWS error and configuration.

---

## 11. Explicit deny wins

The most important IAM rule for troubleshooting:

```text
Explicit Deny
     ↓
Wins over Allow
```

Example:

```text
IAM role policy
  → Allow s3:GetObject

Bucket policy
  → Deny if aws:SecureTransport = false

Request
  → HTTP / insecure transport

Result
  → AccessDenied
```

Adding another `Allow` will not fix the request.

The denied condition must stop being true, or the deny policy must be changed by the security owner.

---

## 12. Conditions are often the hidden failure

Common bucket-policy conditions include:

```text
aws:SecureTransport
aws:SourceVpce
aws:PrincipalArn
aws:PrincipalOrgID
aws:SourceIp
aws:RequestedRegion
s3:x-amz-server-side-encryption
s3:x-amz-server-side-encryption-aws-kms-key-id
```

Example: require requests through a specific VPC endpoint.

```json
"Condition": {
  "StringNotEquals": {
    "aws:SourceVpce": "vpce-0123456789abcdef0"
  }
}
```

This can produce a confusing failure:

```text
IAM policy → Allow
Bucket policy → looks mostly correct
Network → S3 reachable
Request → AccessDenied
```

The missing piece may be the request context: the request came through a different endpoint or path.

---

## 13. Enforcing encryption on uploads

A secure bucket may require uploads to use a specific encryption method.

Example policy pattern:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::company-data-prod/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

If an application uploads without the required encryption header, the request can be denied.

A better troubleshooting question is therefore:

> Is the application sending the encryption configuration required by the bucket policy?

---

## 14. Object ownership and ACLs

Modern S3 designs commonly use **Bucket owner enforced** object ownership, which disables ACL-based access control for the bucket.

Check it:

```bash
aws s3api get-bucket-ownership-controls \
  --bucket company-data-prod
```

Do not automatically troubleshoot every access issue as an ACL issue.

First determine whether ACLs are even part of the bucket's access model.

---

## 15. Production failure scenario — GetObject returns AccessDenied

### Architecture

```text
CI/CD Role
    |
    | s3:GetObject
    v
S3 bucket: company-artifacts-prod
    |
    +--> bucket policy
    |
    +--> Block Public Access
    |
    +--> SSE-KMS
    |
    +--> KMS key policy
```

### Symptom

Deployment pipeline fails:

```text
An error occurred (AccessDenied) when calling the GetObject operation
```

### Step 1 — identify the caller

```bash
aws sts get-caller-identity
```

Confirm the pipeline is actually using the expected role.

### Step 2 — test the exact object

```bash
aws s3api head-object \
  --bucket company-artifacts-prod \
  --key releases/app.tar.gz
```

### Step 3 — inspect IAM

The role has:

```text
Allow s3:GetObject
arn:aws:s3:::company-artifacts-prod/releases/*
```

So IAM appears correct.

### Step 4 — inspect bucket policy

Suppose the bucket contains:

```text
Deny access unless request comes through vpce-0abc...
```

The CI runner is actually using a different VPC endpoint.

Therefore:

```text
IAM Allow
      +
Bucket explicit Deny
      ↓
AccessDenied
```

### Step 5 — check encryption

Suppose the object is SSE-KMS encrypted.

The role also needs the appropriate KMS authorization.

If KMS permission is missing, fixing the bucket policy alone is insufficient.

### Root cause

The failing request did not satisfy the bucket policy's VPC endpoint condition.

### Secure fix

Do not make the bucket public.

Instead:

1. use the approved VPC endpoint/path,
2. update the bucket condition only if the new endpoint is legitimately trusted,
3. ensure the role has only required S3 permissions,
4. ensure KMS authorization is limited to the required key and operation,
5. retest the exact `GetObject` request.

---

## 16. Production failure scenario — PutObject returns AccessDenied

### Symptom

Application upload fails:

```text
AccessDenied: Access Denied
```

IAM says:

```text
Allow s3:PutObject
```

But bucket policy requires:

```text
x-amz-server-side-encryption = aws:kms
```

Application sends no encryption header.

Result:

```text
IAM Allow
      +
Bucket policy explicit Deny
      ↓
AccessDenied
```

### Secure fix

Configure the application to upload using the approved KMS encryption configuration, or use the bucket's approved default encryption model without weakening the security policy unnecessarily.

---

## 17. Production troubleshooting decision tree

```text
S3 AccessDenied
      |
      v
Who is the caller?
aws sts get-caller-identity
      |
      v
What exact operation fails?
GetObject / PutObject / ListBucket / DeleteObject
      |
      v
Is IAM permission present?
      |
   NO +------------------> Fix least-privilege IAM permission
   |
  YES
   |
   v
Check bucket policy
      |
      v
Any explicit Deny?
      |
   YES +-----------------> Identify failing Condition
   |
   NO
   |
   v
Check SCP / permissions boundary / session policy
      |
      v
Is object encrypted with SSE-KMS?
      |
   YES +-----------------> Check KMS key policy + kms permissions
   |
   NO
   |
   v
Check Block Public Access / ACL / ownership if relevant
      |
      v
Retest exact S3 operation
```

---

## 18. AWS CLI command reference

### Identity

```bash
aws sts get-caller-identity
```

### Bucket policy

```bash
aws s3api get-bucket-policy --bucket <bucket>
```

### Public access block

```bash
aws s3api get-public-access-block --bucket <bucket>
```

### Encryption

```bash
aws s3api get-bucket-encryption --bucket <bucket>
```

### Ownership

```bash
aws s3api get-bucket-ownership-controls --bucket <bucket>
```

### Object metadata

```bash
aws s3api head-object --bucket <bucket> --key <key>
```

### List objects

```bash
aws s3api list-objects-v2 --bucket <bucket> --max-items 10
```

### Copy/download

```bash
aws s3 cp s3://<bucket>/<key> ./file
```

### Upload

```bash
aws s3 cp ./file s3://<bucket>/<key>
```

### IAM simulation

```bash
aws iam simulate-principal-policy \
  --policy-source-arn <principal-arn> \
  --action-names s3:GetObject \
  --resource-arns <object-arn>
```

---

## 19. Common mistakes

### Mistake 1 — "IAM allows it, so it must work"

Wrong. Bucket policies and explicit denies can change the result.

### Mistake 2 — Using the bucket ARN for GetObject

`GetObject` targets an object ARN.

### Mistake 3 — Turning off Block Public Access

Never use public access as a shortcut for a private access problem.

### Mistake 4 — Ignoring KMS

SSE-KMS introduces another authorization layer.

### Mistake 5 — Testing only `aws s3 ls`

Listing and object access require different permissions.

### Mistake 6 — Adding more Allow statements blindly

If an explicit Deny is responsible, another Allow will not win.

### Mistake 7 — Ignoring conditions

`aws:SourceVpce`, `aws:SecureTransport`, encryption headers and principal conditions can be the actual root cause.

---

## 20. Interview-ready mental model

When asked:

> "An EC2 role has S3 GetObject permission, but S3 returns AccessDenied. How do you troubleshoot it?"

Answer in this order:

```text
1. Identify the actual caller.
2. Identify the exact S3 API operation and object ARN.
3. Verify IAM identity-based permission.
4. Inspect the bucket policy for explicit Deny and Conditions.
5. Check SCP, permissions boundary and session-policy restrictions.
6. Check Block Public Access / ACL / Object Ownership where relevant.
7. If SSE-KMS is involved, verify KMS key policy and kms:Decrypt/Encrypt permissions.
8. Reproduce the exact operation with AWS CLI.
9. Fix the narrowest failing control and retest.
```

The goal is not:

> "Give the role AdministratorAccess."

The goal is:

> "Find the exact denied statement and make the smallest secure change that permits the required operation."

---

## 21. Beginner takeaway

Remember these five rules:

1. **S3 authorization is multi-layered.**
2. **Explicit Deny beats Allow.**
3. **Bucket actions and object actions use different ARNs.**
4. **SSE-KMS adds KMS authorization to the path.**
5. **Test the exact failing operation instead of assuming all S3 operations are equivalent.**

Mental model:

```text
Caller
  ↓
IAM
  ↓
Bucket Policy
  ↓
Public Access / Ownership controls
  ↓
S3 Object
  ↓
KMS (when applicable)
  ↓
SUCCESS / ACCESS DENIED
```
