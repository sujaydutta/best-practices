# Azure SQL Best Practices (2026 Edition)

Azure SQL Database is a fully managed relational database service that offers high availability, automatic backups, built-in intelligence, and seamless scaling. This guide covers best practices for **Azure SQL Database** (single databases and elastic pools), with a strong focus on the **Hyperscale** service tier for workloads that need to grow to massive scale. It also includes clear guidance on **DTU vs vCore** purchasing models and recommendations for starting small while planning for growth to millions of transactions or users.

These practices align with the broader Azure scalable workloads and observability guidelines.

## 1. Choosing the Right Deployment Option and Service Tier

| Scenario                              | Recommended Option                          | Key Benefits |
|---------------------------------------|---------------------------------------------|--------------|
| New scalable OLTP/HTAP workloads      | **Hyperscale (vCore)**                      | Independent compute/storage scaling, up to 100+ TB, fast restores, readable replicas |
| Variable or bursty workloads          | **Hyperscale Serverless** or **General Purpose Serverless** | Auto-pause, auto-scale compute |
| Predictable, consistent performance   | **Hyperscale Provisioned** or **Business Critical** | High IOPS, local SSD storage |
| Simple small-to-medium apps           | **General Purpose (vCore or DTU)**          | Balanced cost and performance |
| Near full SQL Server compatibility    | **Azure SQL Managed Instance**              | SQL Agent, linked servers, cross-DB queries |

**2026 Recommendation**: For most new applications that may grow significantly, start with **Hyperscale** (vCore model). It supports all workload types and provides the best long-term scalability, performance, and recovery characteristics.

## 2. DTU vs vCore Purchasing Models – Recommendations

Azure SQL offers two purchasing models with different trade-offs in simplicity, flexibility, and cost control.

### DTU Model
- **How it works**: Bundled measure of compute, memory, and I/O into a single "DTU" unit. You choose a performance level (e.g., S3, P11) with fixed resources.
- **Pros**: Simpler to understand and manage; predictable fixed pricing; good for very small or well-understood workloads.
- **Cons**: Less granular control; harder to optimize independently for CPU vs storage vs I/O; limited advanced features (no Serverless in most cases, no Hyperscale).

**When to choose DTU**:
- Very small databases with low and predictable usage.
- Teams new to Azure SQL who want minimal configuration.
- Quick prototypes or dev/test environments where simplicity matters more than fine-tuning.

**Rule of thumb**: If your workload needs **less than ~2 vCores** equivalent, DTU can be simpler (e.g., <200–300 DTUs). Above that, move to vCore.

### vCore Model (Recommended for most production workloads)
- **How it works**: You explicitly choose the number of vCores (logical CPUs), memory, storage, and I/O separately. Available in General Purpose, Business Critical, and Hyperscale tiers.
- **Pros**: Greater flexibility and transparency; independent scaling of compute and storage; access to Serverless compute and Hyperscale; better performance characteristics; Azure Hybrid Benefit savings (up to 55% if you have SQL Server licenses with Software Assurance).
- **Cons**: Slightly more decisions to make initially.

**When to choose vCore**:
- Production workloads of any size.
- Need for Hyperscale, Serverless, or readable replicas.
- When you want to optimize cost/performance (scale compute independently of storage).
- IO-intensive or high-concurrency workloads (Premium/Business Critical or Hyperscale perform better than Standard DTU).

**Migration guideline**: Roughly, **100 DTUs ≈ 1 vCore** (Basic/Standard) or **125 DTUs ≈ 1 vCore** (Premium). However, vCore often delivers better real-world performance for the same nominal resources, especially for I/O-bound workloads.

**Final recommendation for scalable workloads**: Use the **vCore model** with **Hyperscale** tier. It future-proofs your database and avoids painful migrations later.

## 3. Recommendations for Teams Starting with Standard Loads That Can Scale to Millions

Many teams begin with modest workloads (thousands of users/transactions) but need a path to millions without major re-architecture.

**Starting Strategy**:
- Begin with **Hyperscale Serverless (vCore)** or **General Purpose Serverless** with a low minimum (e.g., 0.5–2 vCores) and a reasonable maximum (e.g., 8–16 vCores).
- This gives automatic scaling and auto-pause during idle periods, keeping costs low while the app grows.
- Set storage to start small (e.g., 32–128 GB) — Hyperscale allows storage to grow automatically up to 100+ TB with almost no downtime.

**Scaling Path to Millions**:
- Monitor usage with Azure SQL Insights and Query Performance Insights.
- When sustained load increases:
  - Scale compute up (add vCores) or switch from Serverless to Provisioned for predictable performance.
  - Add readable secondary replicas in Hyperscale to offload reporting and analytics.
  - Use elastic pools if you have multiple related databases sharing resources.
- For very high throughput: Move to **Hyperscale Provisioned** with higher vCores (up to 192 vCores in Premium-series as of 2026) and leverage its page-server architecture for superior I/O and log throughput.

**Practical Starting Configurations**:
- Small/startup load: Hyperscale Serverless, 2 vCores max, 64 GB storage.
- Growing to medium/high load: Scale to 8–16 vCores; add 1–2 readable replicas.
- Millions of transactions/users: Hyperscale Provisioned with 32+ vCores, multiple replicas, and optimized indexing/partitioning.

This approach lets you **start simple and cheap** while having a clear, low-disruption path to massive scale.

## 4. Performance & Scaling Best Practices

- Enable **Automatic Tuning** and follow recommendations from Database Advisor.
- Use readable replicas for read scale-out.
- Optimize queries: covering indexes, batching, parameterized queries, and columnstore where appropriate.
- In Hyperscale, watch log write throughput (typically 100–150 MB/s) for very write-heavy workloads.

## 5. Reliability & High Availability

- Enable **zone-redundant** configuration.
- Set up **failover groups** and geo-replication for disaster recovery.
- Test failovers and point-in-time restores regularly.

## 6. Security Best Practices

- Use **Microsoft Entra ID** authentication and managed identities.
- Disable public network access; use **Private Endpoints**.
- Enable Transparent Data Encryption with customer-managed keys.
- Turn on auditing and Azure Defender for SQL threat detection.

## 7. Observability & Monitoring

- Instrument applications with **OpenTelemetry** and export to **Azure Monitor** / Application Insights.
- Enable **Azure SQL Insights** for detailed telemetry.
- Set alerts on CPU, DTU/vCore usage, log IO, deadlocks, and query duration.

## 8. Cost Optimization

- Use **Reserved Capacity** for steady workloads.
- Leverage **Serverless** for variable usage.
- Right-size regularly with Azure Advisor.
- Monitor and archive old data to control storage growth.

## 9. Development & Operations

- Define databases with **Bicep**.
- Manage schema changes with database projects (SQL Database Projects).
- Implement connection retry logic and transient fault handling in applications.
- Batch operations instead of single-row processing.

## 10. Common Pitfalls to Avoid

- Starting with DTU for workloads expected to grow beyond small scale (migration can be disruptive).
- Using public endpoints or SQL authentication in production.
- Ignoring query tuning even with Automatic Tuning enabled.
- Poor indexing or lack of partitioning on large tables.
- Not utilizing readable replicas for analytics.
- Failing to test scaling and failover scenarios.

---

**Final Advice**:  
For teams building applications that start small but must scale to millions of users or transactions, **Azure SQL Hyperscale in the vCore model** (with Serverless compute initially) offers the best combination of simplicity, performance, and future-proof scalability. The vCore model provides superior flexibility and features compared to DTU, especially as workloads grow.

Start conservatively with low vCore settings and automatic scaling, monitor closely with Azure Monitor, and scale compute/storage/replicas independently as demand increases. Combine this with Entra ID security, Private Endpoints, and OpenTelemetry observability to create a robust, secure, and cost-efficient database foundation.

Regularly review Azure Advisor recommendations and test your scaling strategies to ensure your database grows smoothly with your application.