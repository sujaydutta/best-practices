# AWS RDS Best Practices
> *Covering: Instance Design · High Availability · Security · Performance · Backup & DR · Networking · Monitoring · Cost*

---

## Table of Contents

1. [Instance Design & Sizing](#1-instance-design--sizing)
2. [High Availability & Multi-AZ](#2-high-availability--multi-az)
3. [Read Replicas & Scaling](#3-read-replicas--scaling)
4. [Connection Management & RDS Proxy](#4-connection-management--rds-proxy)
5. [Security](#5-security)
6. [Storage & I/O](#6-storage--io)
7. [Backup, Recovery & DR](#7-backup-recovery--dr)
8. [Performance Tuning](#8-performance-tuning)
9. [Networking & VPC](#9-networking--vpc)
10. [Monitoring & Observability](#10-monitoring--observability)
11. [Maintenance & Upgrades](#11-maintenance--upgrades)
12. [Cost Optimization](#12-cost-optimization)
13. [Engine-Specific Notes](#13-engine-specific-notes)
14. [Quick Reference Checklist](#14-quick-reference-checklist)

---

## 1. Instance Design & Sizing

### 1.1 Right-Sizing from Day One

Choosing the wrong instance class is the most common and expensive RDS mistake. Use the **memory-to-working-set ratio** as your primary sizing signal, not vCPU count.

> **💡 GOLDEN RULE:** Provision enough RAM so that your working set (hot data + indexes) resides almost entirely in memory. Monitor `ReadIOPS` in CloudWatch under load — if it is high and variable, your working set is spilling to disk. Scale up until `ReadIOPS` drops to a small, stable value.

**Instance family selection:**

| Workload | Recommended Family | Notes |
|---|---|---|
| General-purpose OLTP | `db.m7g` (Graviton) | Best price-performance for most workloads |
| Memory-intensive (large working sets) | `db.r7g` / `db.r8g` | Higher RAM-to-vCPU ratio |
| Burstable dev/test | `db.t4g` | Never use `t` class in production |
| High I/O OLTP | `db.r7g` + io2 | Pair memory-optimized with provisioned IOPS |
| Analytics / reporting replica | `db.r7g` + read replica | Separate reads from OLTP writer |

**Key rules:**
- Never run production on `db.t` (burstable) instances — CPU credits can exhaust under sustained load, causing sudden severe degradation
- Graviton (`g`-suffix) instances deliver 20–30% better price-performance than equivalent x86 for most database workloads
- Size for peak, not average — database instances do not scale horizontally mid-request
- Test your working set size with Performance Insights before committing to an instance class

### 1.2 Storage Engine Selection (MySQL / MariaDB)

- Always use **InnoDB** for MySQL — it is the only storage engine that supports crash recovery, Point-In-Time Restore (PITR), and Multi-AZ replication
- MyISAM is not supported for PITR or snapshot restore; using it risks data loss after a crash
- For MariaDB, use **XtraDB** (the InnoDB-compatible engine) for the same reasons

---

## 2. High Availability & Multi-AZ

### 2.1 Always Enable Multi-AZ for Production

Multi-AZ deploys a synchronous standby replica in a different Availability Zone. On failure, RDS automatically promotes the standby — no manual intervention required.

```bash
# Enable Multi-AZ on an existing instance
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --multi-az \
  --apply-immediately
```

**What Multi-AZ protects against:**
- AZ-level hardware failure
- Storage corruption
- Planned maintenance (failover during patching)
- OS-level failures

**What it does NOT protect against:**
- Region-level outages (use cross-region read replicas or Aurora Global Database)
- Accidental data deletion or corruption (use PITR and backups)
- Application-level errors

### 2.2 Failover Behavior and DNS

RDS failover works by updating the instance's DNS CNAME to point to the new primary. This means:

> **⚠️ CRITICAL:** If your application caches DNS, set TTL to **less than 30 seconds**. Stale DNS entries after failover will cause your app to continue hitting a dead endpoint. The AWS suite of drivers handle this automatically.

- Use the **AWS JDBC Driver** (Java), **AWS Advanced Python Driver**, or **AWS ODBC Driver** (.NET) — these use topology-aware failover that reduces failover time to **single-digit seconds**, compared to 30–60 seconds for open-source drivers
- Monitor failover events using **Amazon RDS Event Subscriptions** (SNS notifications or email)
- Use smaller transactions — database recovery relies on transaction logs, so smaller transactions shorten failover recovery time

### 2.3 Multi-AZ with Two Readable Standbys (RDS Optimized Writes)

For Aurora and newer RDS MySQL/PostgreSQL, Multi-AZ with two readable standbys provides:
- Failover in typically **under 35 seconds**
- **2× improved write latency**
- Additional read capacity from the two standbys
- Minor version upgrade downtime of typically **under 1 second**

Use this configuration for latency-sensitive production workloads where the standard Multi-AZ standby is not readable.

---

## 3. Read Replicas & Scaling

### 3.1 Read Replicas for Read Scaling

Read replicas offload read traffic from the primary instance using **asynchronous replication**. They are not a HA mechanism — replica lag means they may be slightly behind the primary.

```bash
# Create a read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica-1 \
  --source-db-instance-identifier mydb-prod \
  --db-instance-class db.r7g.large \
  --availability-zone us-east-1b
```

**Guidelines:**
- Route reporting, analytics, and read-heavy application queries to replica endpoints
- Up to 5 read replicas per RDS instance; up to 15 for Aurora
- Monitor `ReplicaLag` in CloudWatch — lag > 30 seconds indicates the replica may return stale data
- Read replicas can be promoted to standalone instances for DR failover (manual, requires app reconfiguration)
- Cross-region read replicas provide regional DR capability and can be used to migrate to a new region

### 3.2 Aurora vs. RDS for Scaling

For workloads requiring > 5 read replicas or sub-second replica lag, consider **Amazon Aurora**:
- Aurora shares a single distributed storage volume among all replicas — replica lag is typically **under 100ms**
- Aurora Auto Scaling can add/remove read replicas automatically based on CPU or connections
- Aurora Serverless v2 scales compute capacity up/down in increments of 0.5 ACUs (Aurora Capacity Units), ideal for variable workloads

---

## 4. Connection Management & RDS Proxy

### 4.1 The Connection Problem

Every database connection consumes RAM on the DB server and adds overhead. At scale — particularly with Lambda functions, microservices, or multi-threaded apps — **connection storms** (thousands of simultaneous connection requests) can overwhelm the database before a single query runs.

### 4.2 Use RDS Proxy

**RDS Proxy** is a fully managed, highly available connection pooler that sits between your application and RDS. It:
- Maintains a **pool of persistent connections** to the database and multiplexes many application connections onto them
- Reduces failover time by **up to 66%** by preserving application connections during DB failovers (routes to new writer without app reconnection)
- Enforces **IAM authentication** and integrates with **Secrets Manager** — no hardcoded credentials
- Queues connection requests during traffic spikes rather than letting them crash the DB

```bash
# Create an RDS Proxy
aws rds create-db-proxy \
  --db-proxy-name mydb-proxy \
  --engine-family MYSQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:...","IAMAuth":"REQUIRED"}]' \
  --role-arn arn:aws:iam::123456789:role/rds-proxy-role \
  --vpc-subnet-ids subnet-abc subnet-def \
  --vpc-security-group-ids sg-xyz
```

**When to use RDS Proxy:**

| Use Case | Use Proxy? |
|---|---|
| AWS Lambda connecting to RDS | ✅ Always — Lambda creates a new connection per invocation |
| Microservices with many short-lived connections | ✅ Yes |
| Single long-running application server | ⚠️ Optional — evaluate connection churn first |
| Applications with built-in connection pooling (HikariCP, pgBouncer) | ⚠️ Evaluate — proxy adds a hop |
| Analytics / batch jobs with few long connections | ❌ Low value |

### 4.3 RDS Proxy Tuning

Key settings to configure after enabling Proxy:

| Setting | Recommendation |
|---|---|
| `MaxConnectionsPercent` | Start at 70–80% of `max_connections`. Set at least 30% above your peak monitored usage. |
| `MaxIdleConnectionsPercent` | Default is 50% of `MaxConnectionsPercent`. Reduce for memory-constrained DB instances. |
| `IdleClientTimeout` | Default 1800s. Reduce for workloads with bursty, short-lived clients. |
| `ConnectionBorrowTimeout` | Default 120s. Set to your application's query timeout value. |

Monitor these CloudWatch metrics for the proxy:
- `ClientConnections` — incoming connections from your app
- `DatabaseConnections` — connections the proxy holds to the DB
- `MaxDatabaseConnectionsAllowed` — the ceiling; alert when `DatabaseConnections` approaches this

> **⚠️ ANTI-PATTERN:** If `ClientConnections` is consistently lower than `DatabaseConnections`, you have idle DB connections being held open. Lower `MaxIdleConnectionsPercent` or `IdleClientTimeout`.

### 4.4 Application-Level Connection Pooling

If not using RDS Proxy, use a connection pool in your application layer:

- **Java:** HikariCP (fastest, lowest overhead). Set `minimumIdle`, `maximumPoolSize`, and `connectionTestQuery`
- **.NET:** ADO.NET built-in pool or Npgsql pooling for PostgreSQL
- **Python:** SQLAlchemy's `QueuePool`; set `pool_size` and `max_overflow`
- **Node.js:** `pg-pool` for PostgreSQL; `mysql2` pool for MySQL

Always validate connections on borrow (`SELECT 1`) to avoid handing out a dead connection from the pool.

---

## 5. Security

### 5.1 IAM & Access Control

- **Never use root AWS credentials** to manage RDS resources — create individual IAM users/roles with least-privilege policies
- Use **IAM database authentication** for MySQL and PostgreSQL — eliminates long-lived DB passwords and rotates tokens automatically (15-minute validity)
- Apply the **principle of least privilege** at the DB level as well: create separate DB users for each application component with only the permissions they need (e.g., a read-only reporting user, a write-only ingest user)

```sql
-- PostgreSQL: create app-specific users
CREATE USER app_writer WITH PASSWORD 'rotate-via-secrets-manager';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_writer;

CREATE USER app_reader WITH PASSWORD 'rotate-via-secrets-manager';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_reader;
```

### 5.2 Credentials Management

- **Never hardcode database credentials** in application code, environment variables, or config files committed to source control
- Use **AWS Secrets Manager** with automatic rotation enabled for all RDS credentials
- RDS Proxy integrates natively with Secrets Manager — applications authenticate to the proxy via IAM, and the proxy manages credential rotation transparently

```bash
# Enable Secrets Manager rotation for an RDS secret (every 30 days)
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/db \
  --rotation-rules AutomaticallyAfterDays=30
```

### 5.3 Encryption

**At-rest encryption:**
- Enable encryption at creation time using AWS KMS — it cannot be enabled on an existing unencrypted instance (you must snapshot → restore with encryption)
- Use **customer-managed KMS keys (CMK)** for workloads requiring key rotation control, audit trails, or cross-account access
- A CMK used for RDS encryption should be dedicated to that purpose — do not share it across other AWS services
- Encrypted coverage includes: storage, automated backups, read replicas, and snapshots

```bash
# Create encrypted RDS instance
aws rds create-db-instance \
  --db-instance-identifier mydb-prod \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/your-cmk-id \
  ...
```

**In-transit encryption:**
- Force SSL/TLS for all connections by setting `rds.force_ssl = 1` in the parameter group for PostgreSQL and SQL Server
- Use certificate verification in your connection string — do not set `sslmode=require` without `sslrootcert` (verifies endpoint identity)
- For Oracle, use Oracle Native Network Encryption configured in the parameter group

### 5.4 Network Security

- Deploy RDS instances in **private subnets** — never assign a public IP to a production DB instance
- Use **Security Groups** as the primary network control: only allow inbound traffic from your application's Security Group on the DB port (never `0.0.0.0/0`)
- Separate DB Security Groups from application Security Groups — reference the application SG as the source, not its IP range (IPs change; SG membership is stable)
- Enable **VPC Flow Logs** to audit unusual connection patterns

### 5.5 Compliance & Auditing

- Enable **Enhanced Monitoring** and **CloudTrail** for API-level audit logs of all RDS management operations
- Use **AWS Config rules** to enforce compliance: `rds-storage-encrypted`, `rds-snapshots-encrypted`, `rds-instance-public-access-check`, `rds-multi-az-support`
- Use **AWS Security Hub CSPM** to continuously evaluate RDS configurations against security best practices
- For database-level audit logs, enable the appropriate engine feature: `pgaudit` for PostgreSQL, `AUDIT` plugin for MySQL, `audit_log` for SQL Server

---

## 6. Storage & I/O

### 6.1 Storage Types

| Type | Use Case | Characteristics |
|---|---|---|
| **gp3** (General Purpose SSD) | Default for most workloads | 3,000 IOPS / 125 MB/s baseline, up to 16,000 IOPS provisioned; independent throughput/IOPS scaling |
| **io2 Block Express** | Latency-sensitive OLTP, high-write workloads | Up to 256,000 IOPS, 99.999% durability; highest cost |
| **Magnetic (st1)** | Legacy only | Do not use for new deployments |

> **✅ DEFAULT TO gp3.** It replaced gp2 as the recommended general-purpose storage. Unlike gp2 (which ties IOPS to volume size at 3 IOPS/GB), gp3 lets you provision IOPS and throughput independently — you get 3,000 IOPS regardless of volume size, which avoids the gp2 trap of over-provisioning storage just to get IOPS.

### 6.2 Storage Sizing and Autoscaling

- Enable **Storage Autoscaling** for all instances — it expands storage automatically when free space drops below a threshold, preventing outages from full disks
- Storage can only grow, never shrink — set an upper limit (`--max-allocated-storage`) to prevent unbounded growth
- Monitor `FreeStorageSpace` and alert at 20% remaining

```bash
# Enable storage autoscaling with a 500 GB ceiling
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --enable-storage-autoscaling \
  --max-allocated-storage 500
```

### 6.3 I/O Best Practices

- For io2, set Provisioned IOPS at least 10% above your peak observed `WriteIOPS` + `ReadIOPS` — inadequate IOPS lengthens failover recovery time
- Separate data and log I/O where possible (available in SQL Server and Oracle via multi-volume configurations)
- Avoid large, long-running transactions that hold locks and amplify I/O during recovery

---

## 7. Backup, Recovery & DR

### 7.1 Automated Backups

- Set backup retention to **at least 7 days** for production (up to 35 days) — this determines your PITR window
- Schedule backup windows during low-traffic periods, but note that automated backups on Multi-AZ instances occur on the standby, minimizing impact on the primary
- Automated backups are stored in S3 and are replicated across multiple AZs within the region automatically

```bash
# Set 14-day retention during off-peak window
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00"
```

### 7.2 Manual Snapshots

- Take a **manual snapshot before any significant change**: major version upgrade, large schema migration, instance class change
- Manual snapshots persist until you delete them (automated backups expire after the retention period)
- Snapshots of encrypted instances are also encrypted with the same KMS key

### 7.3 Cross-Region Backup & DR Strategy

For RTO/RPO requirements that survive a regional outage:

| DR Tier | RTO | RPO | Approach |
|---|---|---|---|
| **Backup & Restore** | Hours | Hours–days | Cross-region automated backup copy + S3 export |
| **Pilot Light** | 30–60 min | Minutes | Cross-region read replica (promote on failover) |
| **Warm Standby** | Minutes | Seconds | Cross-region replica with pre-scaled instance |
| **Active-Active** | Seconds | Near-zero | Aurora Global Database (replication lag < 1 second) |

```bash
# Enable cross-region backup copy
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --enable-cross-region-backup-copy \
  --backup-replication-source-region us-west-2
```

### 7.4 Testing Your Backups

> **⚠️ A backup you have never tested is not a backup.** Schedule quarterly restore drills:
> 1. Restore latest automated backup to a staging instance
> 2. Validate data integrity and application connectivity
> 3. Measure actual RTO against your DR SLA
> 4. Document any gaps and remediate

---

## 8. Performance Tuning

### 8.1 Enable Performance Insights

Performance Insights is the single most valuable RDS observability tool. It shows **DB Load** (average active sessions by wait event) and identifies the top SQL statements and wait states causing bottlenecks.

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --enable-performance-insights \
  --performance-insights-retention-period 7  # 7 days free; up to 731 days paid
```

Use Performance Insights to answer: *"Is this a CPU problem, an I/O problem, a lock contention problem, or a bad query problem?"*

### 8.2 Parameter Groups

- Never modify the default parameter group — changes apply to all instances using it. Create a **custom parameter group** per environment
- Test parameter changes on a staging instance before applying to production
- Use `pending-reboot` parameters cautiously — schedule changes during a maintenance window
- Key parameters to tune per engine:

**PostgreSQL:**

| Parameter | Recommendation |
|---|---|
| `shared_buffers` | 25% of instance RAM (RDS manages this automatically on most instance types) |
| `work_mem` | Set conservatively (4–16 MB); multiply by `max_connections × sort operations` |
| `autovacuum` | Keep enabled; tune `autovacuum_vacuum_scale_factor` (default 0.2) down to 0.01–0.05 for large, high-churn tables |
| `log_min_duration_statement` | Set to 1000ms to log slow queries for analysis |

**MySQL / Aurora MySQL:**

| Parameter | Recommendation |
|---|---|
| `innodb_buffer_pool_size` | 75–80% of instance RAM (RDS sets this to 75% by default) |
| `innodb_flush_log_at_trx_commit` | Keep at `1` for ACID compliance; do not set to `0` or `2` on primary |
| `slow_query_log` | Enable; set `long_query_time = 1` |
| `max_connections` | Size appropriately — too high wastes memory; use RDS Proxy instead of raising this |

### 8.3 Query Optimization

- Use **EXPLAIN / EXPLAIN ANALYZE** to understand query plans before and after index changes
- Monitor `DatabaseConnections` — a rising connection count without a corresponding rise in throughput indicates connection waiting, not load
- Index foreign keys on the child side of all relationships (MySQL does not create these automatically)
- Use **covering indexes** for high-frequency read queries to eliminate table heap fetches
- Avoid `SELECT *` — only fetch columns you use

### 8.4 Aurora-Specific Performance

- Use **Aurora Parallel Query** for analytical queries that scan large tables — offloads processing to the storage layer
- Use **Aurora Global Database** write forwarding to route writes from a secondary region to the primary without application changes
- For read-heavy workloads, use the **Reader Endpoint** (round-robin across all replicas) rather than individual instance endpoints

---

## 9. Networking & VPC

### 9.1 VPC Layout

```
Region: us-east-1
├── VPC: 10.0.0.0/16
│   ├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)    ← Load balancers, NAT gateways
│   ├── Private App Subnets (10.0.10.0/24, 10.0.11.0/24) ← Application servers
│   └── Private DB Subnets (10.0.20.0/24, 10.0.21.0/24)  ← RDS instances (NO public route)
```

- Always place RDS in **dedicated DB subnet groups** spanning at least 2 AZs
- DB subnets must have no route to the Internet Gateway — no public IP, no NAT route for inbound
- Use **VPC Endpoints** for S3 (if exporting backups) and Secrets Manager to avoid traffic transiting the public internet

### 9.2 Security Group Rules

```
DB Security Group (sg-db):
  Inbound:
    Port 5432 (PostgreSQL) from sg-app-servers   ← App tier SG reference, not IP
    Port 5432 from sg-rds-proxy                  ← If using RDS Proxy
  Outbound:
    (All outbound blocked — DB does not initiate connections)

App Security Group (sg-app-servers):
  Inbound: Port 443 from ALB SG
  Outbound: Port 5432 to sg-db
```

- Never open `0.0.0.0/0` on the DB port — ever
- Reference Security Groups by ID, not by CIDR, to handle dynamic IP assignment in auto-scaling groups

### 9.3 PrivateLink for Cross-Account Access

If application and database live in different AWS accounts, use **AWS PrivateLink** with an NLB and RDS Proxy endpoint — this keeps all traffic on AWS private backbone and avoids VPC peering's transitive routing limitations.

---

## 10. Monitoring & Observability

### 10.1 Core Metrics to Track (CloudWatch)

| Metric | Alert Threshold | What It Indicates |
|---|---|---|
| `CPUUtilization` | > 80% sustained 5 min | Instance undersized or long-running queries |
| `FreeStorageSpace` | < 20% of total | Risk of disk-full outage |
| `FreeableMemory` | < 10% of total RAM | Working set exceeding memory; consider scale-up |
| `ReadIOPS` / `WriteIOPS` | > 90% of provisioned | I/O bottleneck; increase IOPS or optimize queries |
| `DatabaseConnections` | > 80% of `max_connections` | Connection exhaustion risk; add RDS Proxy |
| `ReplicaLag` | > 30 seconds | Replica falling behind; reads may return stale data |
| `FailoverTime` | Any non-zero | Audit event; review failover cause |
| `TransactionLogsDiskUsage` | Rising without bound | Long-running transactions holding WAL/binlog |

### 10.2 Enhanced Monitoring

Enable **Enhanced Monitoring** (granularity: 1–60 seconds) for OS-level metrics not visible in standard CloudWatch: per-process CPU, memory, file system, and I/O — essential for distinguishing DB engine problems from OS-level resource contention.

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --monitoring-interval 15 \
  --monitoring-role-arn arn:aws:iam::123456789:role/rds-monitoring-role
```

### 10.3 Event Subscriptions

Subscribe to RDS events for critical operational alerts:

```bash
aws rds create-event-subscription \
  --subscription-name prod-db-alerts \
  --sns-topic-arn arn:aws:sns:us-east-1:123456789:db-ops-alerts \
  --source-type db-instance \
  --event-categories '["failover","failure","maintenance","notification"]'
```

Key event categories to always subscribe to: `failover`, `failure`, `low storage`, `maintenance`, `recovery`.

### 10.4 Logging

Enable and ship database logs to CloudWatch Logs for retention, searching, and alerting:

- **PostgreSQL:** `postgresql.log` — enable `log_min_duration_statement`, `log_connections`, `log_disconnections`
- **MySQL / Aurora MySQL:** `slow_query_log`, `general_log` (use general log sparingly — high volume), `error_log`
- **SQL Server:** Error log + Agent log

Set CloudWatch log retention policies on all RDS log groups — unset retention means logs are stored indefinitely at cost.

---

## 11. Maintenance & Upgrades

### 11.1 Minor Version Upgrades

- Enable **auto minor version upgrades** — minor versions contain security patches and bug fixes; delaying them increases risk
- Minor upgrades apply during the configured maintenance window with a brief restart (typically < 1 minute on Multi-AZ, where the standby is upgraded first)

### 11.2 Major Version Upgrades

Major version upgrades (e.g., PostgreSQL 14 → 16, MySQL 8.0 → 8.4) require careful planning:

1. **Review release notes** for breaking changes, deprecated features, and changed default parameters
2. **Test application compatibility** on a restored snapshot in staging before touching production
3. **Take a manual snapshot** immediately before the upgrade
4. **Schedule a maintenance window** with your SLA team — even on Multi-AZ, major upgrades involve more downtime than minor ones
5. **Verify parameter group compatibility** — some parameters change between major versions
6. **Use Blue/Green Deployments** (available in RDS) to perform a zero-downtime major upgrade with instant cutover and rollback capability

```bash
# RDS Blue/Green Deployment for major version upgrade
aws rds create-blue-green-deployment \
  --blue-green-deployment-name mydb-upgrade \
  --source mydb-prod-arn \
  --target-engine-version "16.2" \
  --target-db-parameter-group-name mydb-pg16-params
# Test green environment, then:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier bgd-xxx
```

### 11.3 Maintenance Windows

- Set maintenance windows to low-traffic periods (e.g., Sunday 03:00–05:00 UTC)
- Maintenance windows must not overlap with backup windows
- Align all instances in the same stack to the same window to avoid cascading restarts

---

## 12. Cost Optimization

### 12.1 Right-Size Continuously

- Review instance sizing monthly using **AWS Compute Optimizer** and **Cost Explorer DB metrics**
- Idle dev/test instances are a common waste source — schedule automatic stop/start for non-production instances (RDS stops automatically after 7 days if not started; set up EventBridge rules to restart before that)
- Graviton instances (`m7g`, `r7g`) deliver the same or better performance as equivalent x86 at ~15–20% lower cost

### 12.2 Reserved Instances

For production workloads with predictable usage, **Reserved Instances (1-year or 3-year)** provide 30–60% savings over On-Demand. Use Convertible RIs to preserve flexibility for future engine or instance class changes.

### 12.3 Storage Cost

- **gp3 is cheaper than gp2** at the same size — migrate any gp2 volumes to gp3 immediately; you get more IOPS at a lower per-GB cost
- Enable **Storage Autoscaling** with a ceiling to avoid surprise bills from unbounded growth
- Delete old manual snapshots on a schedule — they persist until manually deleted and are billed per GB

### 12.4 Aurora Serverless v2

For workloads with highly variable traffic (dev environments, batch-heavy apps, SaaS tenants), **Aurora Serverless v2** scales from 0.5 to 128 ACUs in 0.5 ACU increments. Minimum capacity of 0 ACUs (paused) eliminates compute cost when idle. Only pay for storage when paused.

---

## 13. Engine-Specific Notes

### 13.1 PostgreSQL

- Enable `pgaudit` extension for detailed audit logging of DML/DDL operations
- Configure **autovacuum aggressively** for tables with high UPDATE/DELETE rates — monitor `pg_stat_user_tables` for `n_dead_tup` and `last_autovacuum`
- Use `pg_stat_statements` extension to identify top SQL by total time, calls, and rows
- When loading bulk data: set `synchronous_commit = off` temporarily (per session), disable autovacuum on target tables, and use `COPY` instead of `INSERT`
- Monitor WAL usage (`TransactionLogsDiskUsage`) — long-running transactions or idle-in-transaction connections block WAL cleanup

### 13.2 MySQL / Aurora MySQL

- InnoDB is the only supported engine for crash-safe PITR — never use MyISAM in RDS
- Monitor `innodb_buffer_pool_read_requests` vs `innodb_buffer_pool_reads` — ratio (buffer pool hit rate) should be > 99%
- Use **binary log retention** carefully — keeping binlogs for long periods increases storage costs; set `binlog retention hours` appropriate to your replication and DR needs
- Avoid implicit commits from DDL in transactions — MySQL's DDL is auto-committed, which can break application-level transactions

### 13.3 SQL Server

- Multi-AZ requires **Full recovery mode** with transaction logging enabled — never set Simple, Offline, or Read-only modes which disable transaction logging
- Use **RDS Event Subscriptions** to monitor Multi-AZ failovers, which are more impactful in SQL Server than other engines
- If you don't need Multi-AZ for specific databases within an instance, use separate non-Multi-AZ instances for those — RDS creates Multi-AZ replicas for **all** databases on the instance
- SQL Server licensing is included in RDS pricing for License Included instances; bring your own license only if you have active Software Assurance

### 13.4 Oracle

- Use **Transparent Data Encryption (TDE)** for at-rest encryption — this is the Oracle-native approach rather than KMS at the storage layer
- Refer to the AWS whitepaper *Best Practices for Running Oracle Database on Amazon Web Services* for detailed guidance
- Test Oracle-specific features (Materialized Views, Advanced Queuing, Partitioning) against your RDS for Oracle version — not all Oracle features are available in RDS

---

## 14. Quick Reference Checklist

Use this checklist when provisioning or reviewing a production RDS instance.

### Instance & HA
- [ ] Instance family is `m7g` or `r7g` (Graviton); `t` class not used in production
- [ ] Multi-AZ enabled
- [ ] Storage type is gp3 or io2 (not gp2 or magnetic)
- [ ] Storage Autoscaling enabled with a defined ceiling
- [ ] Auto minor version upgrades enabled
- [ ] Custom parameter group created (default not used)

### Security
- [ ] Instance is in private subnet with no public IP
- [ ] Security Group allows DB port only from app-tier SG (no `0.0.0.0/0`)
- [ ] Encryption at rest enabled with KMS CMK
- [ ] SSL/TLS forced (`rds.force_ssl = 1` or equivalent)
- [ ] Credentials stored in Secrets Manager with rotation enabled
- [ ] IAM database authentication enabled
- [ ] Deletion protection enabled

### Backup & DR
- [ ] Automated backup retention ≥ 7 days (14+ for production)
- [ ] Backup window does not overlap with maintenance window
- [ ] Cross-region backup copy enabled (for regional DR requirement)
- [ ] Restore drill scheduled (at least quarterly)
- [ ] Pre-change manual snapshot procedure documented in runbooks

### Connection Management
- [ ] RDS Proxy enabled (especially if Lambda, microservices, or high connection churn)
- [ ] Application-level connection pool configured with health check query
- [ ] DNS TTL set to < 30 seconds in app config (or AWS driver in use)

### Monitoring
- [ ] Performance Insights enabled (7-day minimum retention)
- [ ] Enhanced Monitoring enabled (15s granularity)
- [ ] CloudWatch alarms set for CPU, storage, memory, connections, replica lag
- [ ] RDS Event Subscriptions configured for failover and failure events
- [ ] Database logs shipping to CloudWatch Logs with retention policy set

### Cost
- [ ] On gp3 storage (not gp2)
- [ ] Reserved Instance purchased for 1+ year if workload is stable
- [ ] Dev/test instances have scheduled stop/start
- [ ] Old manual snapshots reviewed and pruned