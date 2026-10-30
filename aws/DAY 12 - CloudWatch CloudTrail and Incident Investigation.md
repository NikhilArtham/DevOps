# DAY 12 — CloudWatch, CloudTrail and Incident Investigation

## 1. Why This Topic Matters

In production, knowing that an incident happened is not enough. A DevOps engineer must prove **what happened, when it happened, what AWS resource was involved, which API action caused or changed it, and what operational action should follow**.

CloudWatch and CloudTrail answer different questions:

```text
CloudWatch → What is the system doing?
CloudTrail  → What AWS API activity happened?
```

Together:

```text
Metric / Alarm
      ↓
When did the symptom start?
      ↓
CloudTrail API events
      ↓
What AWS action/change happened?
      ↓
Resource / principal / source
      ↓
Operational action
      ↓
Validate recovery
```

---

## 2. CloudWatch Fundamentals

Amazon CloudWatch is AWS's monitoring and observability service. For DevOps work, the most important concepts are:

- Metrics
- Dimensions
- Namespaces
- Alarms
- Logs and log groups
- Log streams
- Log Insights queries
- Dashboards
- Events/alarms used to trigger operational workflows

### Mental model

```text
Application / AWS service
        ↓
     Metrics
        ↓
     CloudWatch
        ↓
Alarm / Dashboard / Logs
        ↓
Operational decision
```

### Metric

A metric is a time-series measurement.

Examples:

```text
CPUUtilization
NetworkIn
NetworkOut
RequestCount
HTTPCode_ELB_5XX_Count
DatabaseConnections
FreeStorageSpace
```

A metric by itself is not an incident explanation. It is evidence of a system condition.

---

## 3. CloudWatch Metrics and Dimensions

A metric is normally identified by a combination of:

```text
Namespace
Metric name
Dimensions
```

Example:

```text
Namespace: AWS/EC2
Metric: CPUUtilization
Dimension: InstanceId=i-xxxxxxxx
```

For an ALB, dimensions can identify the load balancer and target group.

### Why dimensions matter

If CPU is high across an entire fleet, that is different from one instance having high CPU.

Always ask:

```text
Which resource?
Which metric?
Which dimensions?
Which time window?
```

---

## 4. CloudWatch Alarms

An alarm evaluates a metric against a configured condition.

Example:

```text
CPUUtilization > 80%
for 5 minutes
        ↓
ALARM
```

Important alarm concepts:

- Metric
- Statistic
- Period
- Evaluation periods
- Threshold
- Comparison operator
- Missing data treatment
- Alarm state

### Alarm states

```text
OK
ALARM
INSUFFICIENT_DATA
```

`ALARM` does not automatically mean the root cause has been identified.

It means the configured alarm condition is currently satisfied.

---

## 5. CloudWatch Logs

CloudWatch Logs organizes application/system logs into:

```text
Log Group
   ↓
Log Stream
   ↓
Log Events
```

Example:

```text
/ecs/payment-service
   ↓
container/task stream
   ↓
application log events
```

For incident investigation, identify:

```text
Log group
Time range
Relevant stream(s)
Error pattern
Request/trace identifier, if available
```

---

## 6. CloudWatch Logs Insights

Logs Insights is useful when the log volume is too large for manual inspection.

Basic query:

```text
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

Search for errors:

```text
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100
```

Count errors over time:

```text
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
```

Find HTTP 5xx responses when structured fields are available:

```text
fields @timestamp, status, path
| filter status >= 500
| stats count() by bin(5m), status
| sort @timestamp desc
```

### Query mindset

Do not start by searching random words.

Start with:

```text
Time window
↓
Known symptom
↓
Relevant field/message
↓
Aggregation or correlation
```

---

## 7. AWS CLI: CloudWatch Metrics

List metrics in a namespace:

```bash
aws cloudwatch list-metrics \
  --namespace AWS/EC2
```

Get metric statistics:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxxxxxxx \
  --start-time 2026-10-30T02:00:00Z \
  --end-time 2026-10-30T03:00:00Z \
  --period 300 \
  --statistics Average
```

For modern workflows, `get-metric-data` is also useful when retrieving multiple metrics or time series together.

---

## 8. AWS CLI: CloudWatch Alarms

List alarms:

```bash
aws cloudwatch describe-alarms
```

Inspect a specific alarm:

```bash
aws cloudwatch describe-alarms \
  --alarm-names <alarm-name>
```

Look for:

```text
Metric
Threshold
Period
Evaluation periods
State
State reason
Actions enabled
```

### Important troubleshooting rule

If an alarm fired, capture its **state reason and timestamp**. Do not rely only on the current metric value because the system may have recovered already.

---

## 9. CloudTrail Fundamentals

CloudTrail records AWS API activity.

Typical event information includes:

```text
Event time
Event source
Event name
AWS region
Principal / identity
Source IP
Request parameters
Resources
Response elements
Error code/message
```

Mental model:

```text
Engineer / CI / AWS service
          ↓
       AWS API
          ↓
      CloudTrail
          ↓
       Event
          ↓
Who + What + When + Where + Result
```

CloudTrail is therefore extremely useful when an infrastructure state changed and you need to determine **who or what made the change**.

---

## 10. CloudTrail Event Investigation

Suppose an RDS Security Group rule suddenly changed.

CloudWatch might show:

```text
DatabaseConnections ↓
```

CloudTrail can help answer:

```text
Who modified the security group?
What API call was made?
When did it happen?
From which source?
Was it successful?
```

Potential event names include:

```text
AuthorizeSecurityGroupIngress
RevokeSecurityGroupIngress
ModifyDBInstance
UpdateLoadBalancerAttributes
ModifyListener
CreateRoute
ReplaceRoute
DeleteRoute
PutBucketPolicy
```

The exact event depends on the AWS resource and operation.

---

## 11. CloudTrail Event History vs Trails

### Event history

CloudTrail Event History provides recent management-event visibility in AWS accounts/regions.

Useful for immediate investigation.

### Trail

A CloudTrail trail delivers events to destinations such as S3 and can support longer-term retention, centralized investigation, and organizational auditing depending on the design.

### Operational distinction

```text
Incident happening now
→ start with available CloudTrail event history/search

Long-term audit / centralized investigation
→ use configured CloudTrail trail / log destination
```

Do not assume that every possible event is stored forever in Event History.

---

## 12. AWS CLI: CloudTrail Lookup

Lookup recent events:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyDBInstance
```

Filter by resource name where appropriate:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resource-name>
```

You can also inspect a specific time window and region when supported by the command options.

### Investigation output to capture

```text
Event time:
Event name:
Event source:
AWS region:
User identity:
Source IP:
Resource:
Request parameters:
Error code:
Error message:
```

---

## 13. CloudWatch vs CloudTrail

| Question | CloudWatch | CloudTrail |
|---|---|---|
| Is CPU high? | Yes | No |
| Did request errors increase? | Yes | No |
| Did an alarm fire? | Yes | No |
| Who changed a security group? | Usually no | Yes |
| Which AWS API was called? | No | Yes |
| When did the infrastructure change? | Indirectly | Yes |
| What is the application's log error? | Yes | No |
| What caused a configuration change? | Sometimes indirectly | Often directly |

### Key interview answer

> CloudWatch tells me about the system's **state and behavior**; CloudTrail tells me about **AWS API activity and changes**. During an incident I correlate the two using the same time window and resource.

---

## 14. Incident Investigation Workflow

Use this sequence in production:

```text
1. Establish incident start time
2. Identify affected service/resource
3. Find metric/alarm proving the symptom
4. Expand the time window slightly before the symptom
5. Search CloudWatch Logs for the failure pattern
6. Search CloudTrail for relevant API events
7. Identify principal and exact AWS action
8. Compare change time with symptom time
9. Determine root cause
10. Choose the smallest safe remediation
11. Validate recovery with metrics/logs
12. Document evidence and timeline
```

### Example timeline

```text
10:02  Deployment completes
10:04  ModifySecurityGroupIngress
10:05  Application connection errors increase
10:06  CloudWatch alarm enters ALARM
10:08  Incident declared
10:12  Security group restored
10:14  Error rate returns to baseline
```

The correlation is stronger than simply saying:

```text
"The deployment caused the incident."
```

You need evidence connecting the action to the symptom.

---

## 15. Realistic Production Incident — RDS Connections Suddenly Fail

### Architecture

```text
Application
   ↓
RDS endpoint :5432
   ↓
RDS Security Group
   ↓
PostgreSQL
```

### Symptom

```text
Application database connection errors increase sharply.
CloudWatch DatabaseConnections drops.
Application logs show connection timeouts.
```

### Investigation

#### Step 1 — Prove the symptom

Check:

```text
DatabaseConnections
CPUUtilization
FreeStorageSpace
```

Then inspect application logs for the exact failure time.

#### Step 2 — Check network reachability

```bash
nc -vz <rds-endpoint> 5432
```

Suppose it times out.

#### Step 3 — Investigate AWS changes

Search CloudTrail for:

```text
AuthorizeSecurityGroupIngress
RevokeSecurityGroupIngress
```

Suppose a `RevokeSecurityGroupIngress` event occurred two minutes before the connection failures.

#### Step 4 — Identify the actor

CloudTrail shows:

```text
Principal: CI role
Event: RevokeSecurityGroupIngress
Resource: RDS Security Group
Time: 10:04
```

#### Step 5 — Correlate

```text
10:04 SG rule revoked
        ↓
10:05 TCP 5432 failures
        ↓
10:06 CloudWatch alarm
```

This provides evidence that the security-group change is the likely root cause.

#### Step 6 — Operational action

Do not immediately open:

```text
0.0.0.0/0 → 5432
```

Instead restore the intended least-privilege source Security Group/rule through the approved change path.

#### Step 7 — Validate

```bash
nc -vz <rds-endpoint> 5432
```

Then confirm:

```text
Application errors decrease
CloudWatch alarm returns to OK
DatabaseConnections recover
```

---

## 16. Realistic Production Incident — ALB 5xx Spike After a Change

### Symptom

```text
HTTPCode_ELB_5XX_Count increases
Application 5xx increases
Users report API failures
```

### Investigation path

```text
CloudWatch metric
   ↓
ALB target health
   ↓
CloudWatch application logs
   ↓
CloudTrail API changes
```

Search CloudTrail for recent changes such as:

```text
ModifyListener
ModifyRule
RegisterTargets
DeregisterTargets
```

Then compare:

```text
Change timestamp
vs
5xx spike timestamp
```

If a listener rule changed immediately before the incident and requests are now reaching the wrong target group, the operational fix is to restore the intended routing configuration—not to modify unrelated Security Groups.

---

## 17. Realistic Production Incident — EC2 CPU Spike

### Symptom

```text
CPUUtilization > 90%
```

Do not immediately reboot the instance.

Investigate:

```bash
uptime
top
ps aux --sort=-%cpu | head
```

Then correlate with:

```text
CloudWatch CPU metric
Application logs
Deployment timeline
CloudTrail events
```

Possible causes include:

- Application traffic spike
- Bad deployment
- Run Command execution
- Scheduled workload
- Infinite loop
- Resource leak

The metric identifies the symptom. Logs and CloudTrail can help identify the cause.

---

## 18. Incident Evidence Checklist

For every incident, capture:

```text
Incident start time:
Affected service:
Affected resource ID:
Metric:
Alarm:
Metric timestamp:
Log group:
Relevant log query:
CloudTrail event:
Event time:
Event name:
Principal:
Source IP:
Request parameters:
Root cause:
Remediation:
Validation evidence:
```

This turns troubleshooting into an auditable investigation rather than a sequence of guesses.

---

## 19. Common Mistakes

### Mistake 1 — Treating an alarm as root cause

An alarm proves a condition crossed a threshold. It does not automatically identify why.

### Mistake 2 — Looking only at current state

A recovered system may look healthy now. Historical metric/log/event timestamps are critical.

### Mistake 3 — Ignoring the time window

An event five hours after the incident is weak evidence for causation.

### Mistake 4 — Blaming the latest deployment automatically

Correlation is not proof. Check CloudTrail, logs, metrics and the actual changed resource.

### Mistake 5 — Changing multiple things during an incident

If you change SG + route + application configuration simultaneously, you destroy your ability to identify the actual cause.

### Mistake 6 — Using broad emergency permissions

Do not solve an AccessDenied or connectivity incident with AdministratorAccess or `0.0.0.0/0` unless there is a documented, exceptional reason and approval.

---

## 20. Production Troubleshooting Decision Tree

```text
Incident reported
      ↓
What is the exact symptom?
      ↓
Find CloudWatch metric/alarm
      ↓
When did it start?
      ↓
Check application/system logs
      ↓
What AWS resource is affected?
      ↓
Search CloudTrail around the same time
      ↓
Was there a relevant API change?
      ↓
YES ---------------- NO
 ↓                    ↓
Validate change       Investigate runtime,
against symptom       dependency,
                      capacity and application
 ↓
Choose least-privilege remediation
      ↓
Validate metric + logs + service behavior
      ↓
Document evidence
```

---

## 21. Interview-Ready Mental Model

```text
CloudWatch
= What is happening?

CloudTrail
= What AWS API activity happened?

Logs Insights
= What exactly did the application/system report?

Incident investigation
= Correlate time + resource + symptom + event + action
```

A strong DevOps answer is:

> I first establish the incident timeline using CloudWatch metrics and alarms. I then use Logs Insights to identify the exact runtime failure. In parallel, I search CloudTrail for API activity affecting the same resource and time window. I correlate the event with the symptom, identify the principal and change, apply the smallest safe remediation, and validate recovery using the same metrics and logs that proved the incident.

---

## 22. Beginner Takeaway

Remember these four questions:

```text
CloudWatch:
What is happening?

Logs Insights:
What exactly is the workload reporting?

CloudTrail:
Who/what changed AWS?

Incident response:
What evidence proves the fix worked?
```
