# DAY 11 — RDS HA, Backups, Multi-AZ and Connection Troubleshooting

## Learning Goal

Today is about understanding RDS as both a managed database service and a network endpoint.

The key rule is:

> **A database being `available` does not automatically mean a client can connect to it.**

A connection can fail at DNS, routing, Security Groups/NACLs, the database port, RDS state, TLS, or authentication.

---

## 1. What is Amazon RDS?

Amazon RDS is a managed relational database service. AWS manages much of the underlying infrastructure and operational work such as provisioning, backups, patching options, monitoring integration and high-availability mechanisms.

Common engines include:

```text
PostgreSQL
MySQL
MariaDB
Oracle
SQL Server
Aurora
```

As a DevOps engineer, the important question is not only:

> "Is the RDS instance running?"

It is:

> "Can the application complete the entire path from its source to the database and establish an authenticated database session?"

---

# 2. RDS Connection Mental Model

Use this path whenever an application cannot connect:

```text
Application
    ↓
RDS DNS endpoint
    ↓
DNS resolution
    ↓
Network route
    ↓
Subnet / NACL
    ↓
RDS Security Group
    ↓
Database port
    ↓
RDS database state
    ↓
TLS / authentication
    ↓
Database
```

This is the most important mental model for today's topic.

If you skip layers, you risk changing the wrong thing.

---

# 3. RDS Endpoint

An RDS database exposes a DNS endpoint rather than expecting applications to manage a database IP directly.

Example:

```text
database-1.xxxxx.ap-south-1.rds.amazonaws.com
```

The endpoint resolves to the address used for the current database deployment.

Applications should normally use the RDS endpoint instead of hard-coding an IP address.

Why?

Because AWS can change the underlying infrastructure during operations such as failover.

Mental model:

```text
Application
    ↓
Stable RDS endpoint
    ↓
Current database infrastructure
```

Do not build application configuration around:

```text
Fixed private IP
```

---

# 4. Identify the RDS Endpoint and State

AWS CLI:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id>
```

Useful query:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id> \
  --query 'DBInstances[].{Endpoint:Endpoint.Address,Port:Endpoint.Port,Status:DBInstanceStatus,AZ:AvailabilityZone,MultiAZ:MultiAZ,Engine:Engine}'
```

Important fields:

```text
Endpoint.Address
Endpoint.Port
DBInstanceStatus
AvailabilityZone
MultiAZ
Engine
```

For troubleshooting, record these before changing anything.

---

# 5. Database Ports

Common default ports:

| Engine | Typical Port |
|---|---:|
| PostgreSQL | 5432 |
| MySQL | 3306 |
| MariaDB | 3306 |
| SQL Server | 1433 |
| Oracle | 1521 |

Do not blindly assume the default.

The actual RDS endpoint port is available from:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id> \
  --query 'DBInstances[].Endpoint.Port'
```

A Security Group rule allowing `5432` does not help if the database is configured on another port.

---

# 6. DNS Troubleshooting

First test whether the client can resolve the RDS endpoint.

```bash
dig <rds-endpoint>
```

or:

```bash
nslookup <rds-endpoint>
```

You can also use:

```bash
getent hosts <rds-endpoint>
```

Interpretation:

```text
DNS fails
    ↓
Investigate DNS/resolver/VPC DNS path
```

versus:

```text
DNS resolves
    ↓
Continue to TCP connectivity
```

Important:

**Do not start changing Security Groups when DNS itself is failing.**

---

# 7. Test TCP Connectivity

Once DNS resolves, test the database port.

For PostgreSQL:

```bash
nc -vz <rds-endpoint> 5432
```

For MySQL:

```bash
nc -vz <rds-endpoint> 3306
```

Possible results:

```text
succeeded
```

means the TCP path is reachable.

```text
timed out
```

usually points toward:

```text
Routing
Security Group
NACL
Network path
```

A TCP connection can also be rejected rather than timed out; the exact symptom matters.

---

# 8. Security Group Troubleshooting

RDS commonly has a Security Group allowing traffic from the application Security Group.

Example:

```text
Application SG
      ↓
TCP 5432
      ↓
RDS SG
```

A strong design is:

```text
RDS SG inbound:
Source = Application Security Group
Port = database port
```

rather than:

```text
0.0.0.0/0 → 5432
```

The second configuration unnecessarily exposes the database path.

---

# 9. Security Group Failure Path

Suppose:

```text
Application
   ↓
RDS endpoint
```

DNS works.

But:

```bash
nc -vz <rds-endpoint> 5432
```

times out.

Investigate:

```text
1. Application source
2. Application subnet
3. Route table
4. NACL
5. RDS Security Group
6. RDS port
7. RDS status
```

Do not immediately conclude:

> "RDS is down."

The database may be completely healthy while the network path is blocked.

---

# 10. RDS Subnet Groups

RDS databases are deployed into an RDS DB subnet group.

Inspect it with:

```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name <subnet-group>
```

A DB subnet group contains subnets from multiple Availability Zones.

Mental model:

```text
VPC
 │
 ├── AZ-a
 │    └── DB subnet
 │
 └── AZ-b
      └── DB subnet
```

This multi-AZ subnet placement is important for resilient database deployments.

---

# 11. Multi-AZ — What Problem Does It Solve?

Multi-AZ is primarily an **availability and failover** feature.

Conceptually:

```text
Primary DB
   │
   │ synchronous replication / managed HA
   ↓
Standby in another AZ
```

If the primary database deployment experiences an eligible failure, RDS can perform a managed failover.

The application should reconnect through the RDS endpoint rather than depending on a fixed IP.

---

# 12. Multi-AZ Is Not the Same as a Read Replica

This distinction is important in interviews.

### Multi-AZ

Primary purpose:

```text
High availability
Failover
```

### Read Replica

Primary purpose:

```text
Read scaling
Replication-based workloads
```

Do not answer:

> "Multi-AZ is used to scale reads."

That is the wrong mental model for standard RDS Multi-AZ DB instances.

---

# 13. Inspect Multi-AZ

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id> \
  --query 'DBInstances[].{Status:DBInstanceStatus,MultiAZ:MultiAZ,AZ:AvailabilityZone}'
```

Example:

```text
Status: available
MultiAZ: true
AZ: ap-south-1a
```

Interpret this as:

```text
Current primary location = ap-south-1a
HA configuration = enabled
```

Do not interpret `MultiAZ=true` as:

> "The application is automatically connected to two databases simultaneously."

---

# 14. RDS Backups

Backups solve a different problem from Multi-AZ.

Think:

```text
Multi-AZ
→ Availability / failover

Backups
→ Data recovery
```

Backups help with scenarios such as:

```text
Accidental deletion
Bad data change
Point-in-time recovery
Database recovery
```

Multi-AZ does **not** replace backups.

---

# 15. Backup Retention

Inspect backup configuration:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id> \
  --query 'DBInstances[].{BackupRetention:BackupRetentionPeriod,BackupWindow:PreferredBackupWindow}'
```

Important fields:

```text
BackupRetentionPeriod
PreferredBackupWindow
```

The retention period determines how long automated backups are retained according to the database configuration.

---

# 16. Automated Backups vs Manual Snapshots

### Automated backups

Useful for:

```text
Point-in-time recovery
Automated recovery workflow
Retention-based recovery
```

### Manual snapshots

Useful for:

```text
Explicit recovery points
Pre-change snapshots
Longer-term preservation/use cases
```

Mental model:

```text
Automated backup
→ continuous recovery capability within retention

Manual snapshot
→ explicitly created recovery point
```

---

# 17. Backup Is Not a Connection Fix

Suppose:

```text
Application → RDS
```

and the application gets:

```text
Connection timeout
```

Turning on backups will not fix:

```text
Blocked Security Group
Wrong port
Broken route
DNS failure
NACL denial
```

This is a common troubleshooting mistake: changing an availability/recovery feature when the failure is actually networking.

---

# 18. Connection Failure Decision Tree

Use this sequence:

```text
Can DNS resolve the RDS endpoint?
        │
        ├── NO → DNS layer
        │
        └── YES
             ↓
Can TCP reach the RDS port?
        │
        ├── NO → Route / NACL / SG / network
        │
        └── YES
             ↓
Is RDS available?
        │
        ├── NO → RDS service/database state
        │
        └── YES
             ↓
Does database authentication succeed?
        │
        ├── NO → Credentials / TLS / DB authorization
        │
        └── YES
             ↓
Application/database interaction
```

This is much stronger than simply saying:

> "Check the Security Group."

---

# 19. AWS CLI Troubleshooting Commands

### RDS instance

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-id>
```

### RDS subnet group

```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name <subnet-group>
```

### Security Groups

```bash
aws ec2 describe-security-groups \
  --group-ids <sg-id>
```

### DNS

```bash
dig <rds-endpoint>
```

### TCP

```bash
nc -vz <rds-endpoint> <port>
```

### Route

```bash
ip route get <destination-ip>
```

The objective is to identify the **first failing layer**, not collect random commands.

---

# 20. Realistic Production Failure

Architecture:

```text
Application EC2
     │
     │ TCP 5432
     ↓
RDS PostgreSQL
```

Incident:

```text
Application cannot connect to database.
Connection timeout.
```

### Step 1 — Check RDS

```bash
aws rds describe-db-instances \
  --db-instance-identifier prod-db \
  --query 'DBInstances[].{Status:DBInstanceStatus,MultiAZ:MultiAZ,Endpoint:Endpoint.Address,Port:Endpoint.Port}'
```

Result:

```text
Status = available
MultiAZ = true
Port = 5432
```

RDS itself appears healthy.

### Step 2 — Check DNS

```bash
dig prod-db.xxxxx.ap-south-1.rds.amazonaws.com
```

DNS resolves.

### Step 3 — Test TCP

```bash
nc -vz prod-db.xxxxx.ap-south-1.rds.amazonaws.com 5432
```

Result:

```text
Connection timed out
```

### Step 4 — Inspect Security Group

RDS Security Group contains:

```text
TCP 5432
Source: old-application-sg
```

But the application was recently migrated to:

```text
new-application-sg
```

### Root cause

The database is healthy.

Multi-AZ is healthy.

Backups are healthy.

DNS is healthy.

The failure is:

```text
RDS Security Group
```

### Least-privilege fix

Allow:

```text
RDS SG
TCP 5432
Source = new application SG
```

Do **not** fix it with:

```text
0.0.0.0/0 → 5432
```

Then retest:

```bash
nc -vz <rds-endpoint> 5432
```

and finally validate the actual database connection.

---

# 21. Production Troubleshooting Checklist

When an RDS connection fails:

```text
[ ] Identify RDS endpoint
[ ] Identify database engine
[ ] Identify actual database port
[ ] Check DNS resolution
[ ] Identify client source subnet
[ ] Check route path
[ ] Check NACL
[ ] Check RDS Security Group
[ ] Check RDS status
[ ] Check Multi-AZ configuration
[ ] Check backup retention
[ ] Test TCP with nc
[ ] Test database authentication
[ ] Check TLS if applicable
[ ] Check application connection configuration
```

Always preserve the evidence before changing anything.

---

# 22. Common Mistakes

### Mistake 1
"RDS is available, so networking is fine."

Wrong.

`available` does not prove the client can reach the database.

### Mistake 2
"Enable Multi-AZ to fix connection timeouts."

Wrong.

Multi-AZ improves availability; it does not fix a blocked port.

### Mistake 3
"Enable backups to fix the outage."

Wrong.

Backups solve recovery problems, not connectivity problems.

### Mistake 4
"Open RDS port to the internet."

Dangerous.

Prefer application-SG-to-RDS-SG access.

### Mistake 5
"Use the RDS IP in the application."

Fragile.

Use the managed RDS endpoint.

### Mistake 6
"The Security Group looks correct, so networking is fine."

Not enough.

Verify:

```text
actual source
actual destination port
route
NACL
DNS
RDS state
```

---

# 23. Interview-Ready Mental Model

Remember:

```text
RDS CONNECTION

Endpoint
   ↓
DNS
   ↓
Route
   ↓
NACL
   ↓
Security Group
   ↓
Port
   ↓
RDS State
   ↓
TLS/Auth
   ↓
Database
```

And separately:

```text
Multi-AZ
→ Availability / Failover

Backups
→ Recovery / PITR

Read Replica
→ Read Scaling
```

If you can explain those two diagrams clearly in an interview, you understand the core of today's topic.

---

# Beginner Takeaway

If you remember only five things:

1. **RDS endpoint is the application's database address.**
2. **DNS success does not guarantee TCP connectivity.**
3. **Security Groups control database network access.**
4. **Multi-AZ is for high availability/failover.**
5. **Backups are for recovery, not connectivity.**

The DevOps troubleshooting rule is:

> **Find the first layer that fails, fix only that layer, and retest the exact connection.**