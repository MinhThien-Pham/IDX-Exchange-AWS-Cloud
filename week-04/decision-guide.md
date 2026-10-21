# Week 4 — Storage & Database Decision Guide

| Service | What it is | Access pattern | Scope / durability | Use this when... |
|---|---|---|---|---|
| **S3** | Object storage. Files (objects) live in buckets and are reached by key over HTTPS. | Whole-object PUT/GET through an API. Not a mounted disk. | Regional. Stored across at least 3 AZs, 11 nines of durability. | ...you need to store files, backups, logs, static website assets or data-lake data at any scale, and pay only for what you store. |
| **EBS** | Block storage. A virtual hard drive attached to one EC2 instance. | Mounted as a disk with a filesystem. Low-latency random reads/writes. | Lives in one AZ. Usually attached to one instance at a time. Snapshots go to S3. | ...one EC2 instance needs a boot volume or a fast local disk for an OS, an application or a self-managed database. |
| **EFS** | Managed NFS file system. | Mounted by many Linux instances (and containers/Lambda) at the same time. | Regional. Spans multiple AZs and grows and shrinks automatically. | ...several instances or containers need to read and write the same files at once, e.g. shared uploads or content behind an Auto Scaling Group. |
| **RDS** | Managed relational database (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server). | SQL with joins, transactions, and a fixed schema. | Single-AZ, or Multi-AZ with a synchronous standby for automatic failover. AWS handles backups and patching. | ...the data is relational and needs SQL, joins, transactions or ad-hoc queries, like PropertyLite's `rets_property` listings. |
| **DynamoDB** | Managed NoSQL key-value and document database. | Get/put by partition key (plus an optional sort key). Single-digit-millisecond latency at any scale. | Regional. Replicated across 3 AZs. Serverless, with on-demand or provisioned capacity. | ...you look items up by a known key at high scale with predictable low latency and no servers to manage, like fetching one listing by `L_ListingID`. |

## How I choose

1. **Is it files or records?** For files, use S3 (or EFS if it must be a shared mounted filesystem). For records, use a database.
2. **Does one server need a disk?** Use EBS. **Do many servers need the same disk?** Use EFS.
3. **Do I need SQL, joins or transactions across tables?** Use RDS. **Do I only look things up by key?** Use DynamoDB.
4. **Cost and lifecycle:** keep data that is rarely read in S3 Standard-IA or Glacier with a lifecycle rule. Glacier has a 90-day minimum storage charge, so don't archive data that will be deleted soon.

## S3 storage classes (Lab 4.1)

| Class | Use this when... |
|---|---|
| Standard | Data is read often. |
| Standard-IA | Data is read less than about once a month but must come back instantly. 30-day minimum. |
| Glacier Flexible Retrieval | Archive data. Retrieval takes minutes to hours. 90-day minimum. |

## What I built this week

### Lab 4.1 — S3 with a lifecycle rule

Created `my-training-bucket-minh`, uploaded `week-03/runbook.md`, and added the lifecycle rule `archive-after-30-days` (all objects → Glacier Flexible Retrieval after 30 days, status `Enabled`). `aws s3 ls` showed the uploaded file.

### Lab 4.2 — RDS MySQL with the real `rets_property` data

- **Instance:** `training-db`, MySQL 8.0.46, `db.t4g.micro`, 20 GB gp2, single-AZ, **not publicly accessible**.
- **Network:** the RDS security group allows port 3306 only from the security group of a temporary EC2 client in the same VPC (source = security group ID, not an IP). This is the pattern from Debug Lab 4.4. Nothing on the internet can reach the database.
- **Import:** loaded the raw `rets_property.sql` dump as-is from the EC2 client. I compressed it first (629 MB → 150 MB) so the upload took 14 seconds. The import took about 22 minutes and loaded **55,212 rows and 126 columns**.
- **PII cleanup:** dropped all 17 identifying columns (agent names, emails, phones, addresses, parcel IDs, coordinates, photos) **before running any query**. `DESCRIBE rets_property` showed 109 columns and none of the 17.
- **Synthetic IDs:** replaced the real MLS `L_ListingID` with `R100001`–`R155212`, ordered by `id` so the mapping is the same on every re-run. All 55,212 IDs are unique and start with `R1`.

**A problem I hit in step 10:** the last statement, `ALTER TABLE rets_property ADD PRIMARY KEY (L_ListingID)`, failed with `ERROR 1068: Multiple primary key defined`. The dump already defines `PRIMARY KEY (id)`, and `id` is `AUTO_INCREMENT`, which MySQL requires to stay indexed. So I couldn't just drop the old key. The fix was to swap both in one statement:

```sql
ALTER TABLE rets_property
  DROP PRIMARY KEY,
  ADD UNIQUE KEY uk_id (id),
  ADD PRIMARY KEY (L_ListingID);
```

**Aggregate check** (top 10 cities by listing count):

| L_City | COUNT(*) | AVG(L_SystemPrice) |
|---|---:|---:|
| Los Angeles | 4015 | 2,639,425 |
| San Diego | 2198 | 1,151,293 |
| San Jose | 1047 | 1,387,793 |
| Irvine | 723 | 2,058,264 |
| Palm Desert | 598 | 739,989 |
| Long Beach | 597 | 1,008,757 |
| Palm Springs | 580 | 885,824 |
| Riverside | 539 | 846,260 |
| Oakland | 483 | 798,901 |
| Indio | 465 | 540,047 |

**Handling the raw file:** the raw dump still has real people's contact details, so I treated it that way. It is in `.gitignore` and was never committed. The compressed copy lived outside the repo and was deleted afterwards. The copy on the EC2 client was deleted before the instance was terminated. The master password was generated randomly and kept outside the repo.

**Cost note:** RDS reports this version as `8.0.46-eol`. MySQL 8.0 is past standard support on RDS, so AWS adds an Extended Support charge (about $0.10 per vCPU-hour, which is about $0.20/hour on a 2-vCPU `db.t4g.micro`) on top of the instance price. The lab requires 8.0 for `ROW_NUMBER()`, but MySQL 8.4 LTS supports it too and would avoid that charge for anything long-lived.

### Lab 4.3 — DynamoDB

Created the `Properties` table (partition key `L_ListingID`, on-demand billing), then loaded one row from the **cleaned** RDS table (`R100234`, San Leandro, 4 beds, 2 baths, 1,603 sqft, $799,000, Active). `get-item` returned it with the same fields PropertyLite returns.

### Cleanup

Deleted the DynamoDB table, deleted the RDS instance without a final snapshot, removed the lifecycle rule and the object while it was still in Standard (Glacier has a 90-day minimum charge), and terminated the EC2 client and removed both security groups. The empty bucket is left in place. Empty buckets cost nothing.
