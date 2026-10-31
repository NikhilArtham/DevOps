# DAY 11 — October Production Troubleshooting Lab & Monthly Mock Interview

## 1. What today's session is

This is not a new command-learning day. It is a **production troubleshooting and interview-performance day** combining Linux and AWS.

The goal is to prove that you can:

```text
Symptom
  ↓
Scope the incident
  ↓
Identify the failing layer
  ↓
Collect evidence
  ↓
Form a hypothesis
  ↓
Test the hypothesis
  ↓
Apply the smallest safe fix
  ↓
Validate recovery
  ↓
Explain prevention
```

Today's rule: **answer before looking at notes.**

---

## 2. Interview answer standard

A strong production answer should normally contain five parts:

1. **Symptom** — what the users/system reported.
2. **Evidence** — the command, metric, log or AWS event that proves the state.
3. **Root cause** — the specific layer that failed.
4. **Remediation** — the smallest safe operational action.
5. **Validation/prevention** — how you prove recovery and reduce recurrence.

Avoid answers such as:

> "I will check the logs and restart the service."

That is not troubleshooting. It is a list of actions without a hypothesis.

A stronger answer is:

> "I would first establish whether the failure is DNS, network, listener, service or application-level. I would collect evidence for the affected time window, identify the first meaningful failure, test the suspected layer, make the smallest safe change, and then validate the same path that originally failed."

---

# 3. Production Troubleshooting Lab

## Scenario A — Linux service failure

### Incident

Users report that an internal API is returning errors after a deployment.

Architecture:

```text
Client
  ↓
Load Balancer
  ↓
EC2
  ↓
systemd service
  ↓
Application
```

Symptoms:

```text
HTTP 503/connection failures
EC2 is running
Deployment completed
```

### Your investigation

Start from the host:

```bash
systemctl status <service>
systemctl is-active <service>
systemctl is-enabled <service>
```

Inspect the service definition:

```bash
systemctl cat <service>
systemctl show <service> -p User -p Group -p ExecStart -p ActiveState -p SubState
```

Inspect the incident window:

```bash
journalctl -u <service> --since "30 minutes ago" --until "now" --no-pager
journalctl -p err -b --since "30 minutes ago" --no-pager
```

Check the listener:

```bash
ss -lntp
ss -lntp | grep ':8080'
```

### Possible root causes

Do not assume the service itself is the root cause. Test whether the failure is:

```text
systemd configuration
application crash
missing environment variable
permission problem
port already occupied
wrong bind address
dependency failure
DNS/network dependency
bad deployment artifact
```

### Production-quality answer

Your final explanation should be:

```text
Symptom:
Evidence:
Failed layer:
Root cause:
Safe remediation:
Validation:
Prevention:
```

---

# 4. Scenario B — AWS connectivity incident

### Incident

An application running on EC2 cannot connect to an RDS database.

Architecture:

```text
EC2
 ↓
RDS endpoint
 ↓
RDS database
```

Application error:

```text
connection timeout
```

### Investigation order

Do not immediately change Security Groups.

Start with DNS:

```bash
dig <rds-endpoint>
getent hosts <rds-endpoint>
```

Then test TCP connectivity:

```bash
nc -vz <rds-endpoint> 5432
```

Check the EC2 network placement and route path:

```text
EC2
 ↓
Subnet
 ↓
Route table
 ↓
Network controls
 ↓
RDS subnet
 ↓
RDS Security Group
 ↓
5432
```

Check the database state and configuration using AWS CLI:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id> \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,Port:Endpoint.Port,MultiAZ:MultiAZ,AZ:AvailabilityZone}'
```

### Strong interview reasoning

If DNS resolves but TCP times out:

```text
DNS is probably not the failing layer.
```

Then investigate:

```text
route
NACL
Security Group
RDS state
```

If TCP connects but authentication fails:

```text
network path is working
→ investigate credentials/authentication/database authorization
```

This distinction is important: **a timeout and an authentication failure are not the same incident.**

---

# 5. Scenario C — AWS change caused a production incident

### Incident

At 10:15 AM, an ALB-backed application starts returning 503 responses.

At 10:13 AM, someone changed an AWS resource configuration.

Your job is to determine whether the change caused the incident.

### CloudWatch evidence

Check the affected time window for:

```text
RequestCount
HTTPCode_ELB_5XX_Count
HTTPCode_Target_5XX_Count
TargetResponseTime
HealthyHostCount
UnHealthyHostCount
```

The important question is not:

> "Did the alarm fire?"

It is:

> "What changed in the system when the alarm condition began?"

### CloudTrail investigation

Search CloudTrail for the affected resource and time window.

Useful CLI pattern:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=<API_ACTION> \
  --max-results 50
```

You can also investigate by resource name or inspect CloudTrail records in the console/CloudTrail Lake where available.

Record:

```text
Event time:
Event name:
AWS service:
Principal:
Source IP:
Resource:
Request parameters:
Error code:
```

### Correlation test

Build a timeline:

```text
10:12 — application healthy
10:13 — AWS API change
10:14 — target health changes
10:15 — 503 alarm fires
10:16 — users report failures
```

A timeline like this is strong evidence that the AWS change is related, but still verify the actual causal relationship.

### Operational action

If the change is proven to be the cause:

```text
Stop further changes
→ preserve evidence
→ identify exact configuration delta
→ rollback/revert through approved process
→ validate target health
→ validate application traffic
→ document root cause
```

Do not blindly make unrelated Security Group or routing changes just because the application is unavailable.

---

# 6. Monthly Mock Interview — October

## Rules

Before opening notes:

- Answer each question aloud.
- Keep first answers to **30–60 seconds**.
- For troubleshooting scenarios, explain your investigation order rather than listing random commands.
- If you miss a question, write the technically correct answer and one production example.
- Re-answer every weak question until it is concise and defensible.

### Scoring

For each question:

```text
2 = Correct, concise and production-ready
1 = Partially correct / missing an important layer
0 = Incorrect or guessed
```

For troubleshooting scenarios, also score:

```text
+1 Evidence-based reasoning
+1 Safe remediation and validation
```

Target:

```text
≥ 85%  → Strong
70–84% → Needs targeted revision
< 70%  → Revisit fundamentals before moving on
```

---

## Round 1 — Linux fundamentals

### Q1. A Linux service is active but users cannot connect. What do you check?

Expected thinking:

```text
service state
→ listener/socket
→ bind address
→ local firewall
→ network path
→ dependency
→ application logs
```

Do not assume `systemctl status` being active means the application is reachable.

### Q2. What is the difference between `df` and `du`?

Your answer must explain what happens when `df` reports a full filesystem but `du` cannot account for the usage.

### Q3. A process is consuming 100% CPU. How do you investigate safely?

Your answer should include:

```text
top/ps
→ PID
→ command/process context
→ logs/application behavior
→ impact assessment
→ remediation
```

Explain why `kill -9` should not be your first action.

### Q4. Why can a systemd service fail immediately after a successful deployment?

Cover at least:

```text
configuration
permissions
environment
missing files
port conflicts
dependencies
application crash
```

### Q5. A cron job works manually but fails every night. Why?

Discuss:

```text
PATH
working directory
user
environment variables
permissions
credentials
absolute paths
logging
```

---

# Round 2 — AWS troubleshooting

### Q6. An EC2 instance cannot reach the internet. What is your investigation order?

Expected layers:

```text
instance
→ subnet
→ route table
→ NAT/IGW
→ NACL
→ Security Group
→ DNS
→ destination
```

### Q7. An ALB returns 503. What does that tell you?

Do not simply say "the server is down."

Explain target health, listener/routing, target group and application reachability.

### Q8. An application gets `AccessDenied` when calling S3. How do you investigate?

Cover:

```text
caller identity
→ IAM identity policy
→ bucket policy
→ explicit deny
→ SCP/boundary/session constraints
→ KMS if encryption is involved
→ resource/object ARN
→ conditions
```

### Q9. An EC2 instance cannot connect to RDS on port 5432. What evidence separates DNS, network and authentication failures?

A strong answer distinguishes:

```text
DNS resolution
TCP connection
application/database authentication
```

### Q10. How do you investigate whether an AWS change caused an incident?

Your answer must include both:

```text
CloudWatch
```

and:

```text
CloudTrail
```

Explain the difference between **observing system behavior** and **proving an AWS API change occurred**.

---

# Round 3 — Production reasoning

### Q11. Users report intermittent failures. What is your first response?

Do not immediately restart services.

Explain how you establish:

```text
scope
start/end time
affected users/resources
error pattern
recent changes
metrics/logs
```

### Q12. What makes a troubleshooting hypothesis strong?

A strong hypothesis should:

```text
explain the symptom
predict observable evidence
be testable
avoid unnecessary changes
```

### Q13. You find a recent configuration change. Is that automatically the root cause?

Answer no, then explain why temporal correlation is evidence but not proof by itself.

### Q14. During an incident, why should you avoid making many changes at once?

Explain:

```text
loss of causal evidence
larger blast radius
harder rollback
unclear root cause
```

### Q15. What should your final incident summary contain?

Use:

```text
Impact
Timeline
Detection
Evidence
Root cause
Remediation
Validation
Prevention
```

---

# 7. Weak-question loop

For every question scored 0 or 1, create this record:

```text
Question:
My answer:
What I missed:
Technically correct explanation:
Production example:
Final concise answer:
```

Then repeat the same question aloud.

The target is not memorizing the written answer. The target is being able to explain the reasoning **without notes**.

---

# 8. Final interview mental model

For almost every production troubleshooting question, think:

```text
WHAT BROKE?
    ↓
WHO/WHAT IS AFFECTED?
    ↓
WHEN DID IT START?
    ↓
WHAT CHANGED?
    ↓
WHICH LAYER FAILED?
    ↓
WHAT EVIDENCE PROVES IT?
    ↓
WHAT IS THE SMALLEST SAFE FIX?
    ↓
DID THE SYSTEM RECOVER?
    ↓
HOW DO WE PREVENT RECURRENCE?
```

### Beginner takeaway

A DevOps engineer is not judged by how many commands they can remember.

They are judged by whether they can move from **symptom → evidence → root cause → safe action → validation** without guessing.
