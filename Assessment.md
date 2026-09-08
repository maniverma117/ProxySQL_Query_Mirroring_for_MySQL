Absolutely. Since this is **800 GB of MySQL data, 1 TB current storage, Cloud SQL → AWS EC2 self-managed MySQL 8.0, with DMS full-load + CDC and then AWS replicas**, I would first build a **production assessment pack**.

The important principle is:

> **Don't copy GCP's current MySQL configuration blindly.** Measure the workload first, then size CPU/RAM/EBS/MySQL buffers/redo/binlog based on real production behavior.

AWS DMS supports Google Cloud MySQL 8.0 as a source, and CDC requires the appropriate binary-log configuration. ([AWS Documentation][1])

---

# 1. First: collect the complete GCP baseline

I would collect these categories:

```text
A. Database inventory
B. Table sizes
C. Index sizes
D. Storage growth
E. Connections
F. QPS / TPS
G. Read vs write workload
H. Slow queries
I. CPU
J. Memory
K. Disk IOPS
L. Disk throughput
M. Network
N. InnoDB health
O. Buffer pool efficiency
P. Redo/log pressure
Q. Binary-log generation
R. Current replication health
S. MySQL configuration
T. DMS prerequisites
```

---

# 2. Basic MySQL version / identity

Run:

```sql
SELECT
    VERSION() AS mysql_version,
    @@hostname AS hostname,
    @@server_uuid AS server_uuid,
    @@server_id AS server_id,
    @@port AS port,
    @@datadir AS datadir,
    @@default_storage_engine AS default_storage_engine,
    @@character_set_server AS character_set_server,
    @@collation_server AS collation_server,
    @@time_zone AS time_zone;
```

Also:

```sql
SHOW VARIABLES LIKE 'version%';
```

---

# 3. Database size

This is one of the most important queries.

```sql
SELECT
    table_schema AS database_name,
    COUNT(*) AS table_count,
    ROUND(SUM(data_length) / 1024 / 1024 / 1024, 2) AS data_gb,
    ROUND(SUM(index_length) / 1024 / 1024 / 1024, 2) AS index_gb,
    ROUND(
        SUM(data_length + index_length)
        / 1024 / 1024 / 1024,
        2
    ) AS total_gb
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
GROUP BY table_schema
ORDER BY total_gb DESC;
```

This tells us:

```text
database
tables
data
indexes
total
```

---

# 4. Total database size

For a single overall number:

```sql
SELECT
    ROUND(
        SUM(data_length + index_length)
        / 1024 / 1024 / 1024,
        2
    ) AS total_database_gb
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
);
```

You expect roughly:

```text
~800 GB
```

But note that `information_schema` table sizes and Cloud SQL provisioned storage aren't necessarily identical. The **1 TB allocated storage vs ~800 GB logical table data** should be treated separately.

---

# 5. Largest tables

This is extremely important for DMS.

```sql
SELECT
    table_schema,
    table_name,
    engine,
    table_rows,

    ROUND(data_length / 1024 / 1024 / 1024, 2) AS data_gb,

    ROUND(index_length / 1024 / 1024 / 1024, 2) AS index_gb,

    ROUND(
        (data_length + index_length)
        / 1024 / 1024 / 1024,
        2
    ) AS total_gb

FROM information_schema.tables

WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)

ORDER BY
    (data_length + index_length) DESC

LIMIT 100;
```

For your 800 GB database, I particularly want to know if you have:

```text
1 × 300 GB table
2 × 100 GB tables
1000 × small tables
```

because DMS behaves very differently depending on the table distribution.

---

# 6. Storage engine

You want to know whether everything is InnoDB.

```sql
SELECT
    engine,
    COUNT(*) AS tables,
    ROUND(
        SUM(data_length + index_length)
        / 1024 / 1024 / 1024,
        2
    ) AS total_gb
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
GROUP BY engine
ORDER BY total_gb DESC;
```

Ideally:

```text
InnoDB    ~800 GB
```

If you have significant:

```text
MyISAM
MEMORY
CSV
```

I want to know before migration.

---

# 7. Number of tables

```sql
SELECT
    COUNT(*) AS total_tables
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
AND table_type = 'BASE TABLE';
```

---

# 8. Partitioned tables

Very important for DMS.

```sql
SELECT
    table_schema,
    table_name,
    COUNT(*) AS partitions
FROM information_schema.partitions
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
AND partition_name IS NOT NULL
GROUP BY table_schema, table_name
ORDER BY partitions DESC;
```

AWS DMS has specific limitations around partitioned MySQL tables and CDC/DDL, so I would specifically inventory them. ([AWS Documentation][2])

---

# 9. Foreign keys

```sql
SELECT
    table_schema,
    COUNT(*) AS foreign_keys
FROM information_schema.referential_constraints
WHERE constraint_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
GROUP BY table_schema
ORDER BY foreign_keys DESC;
```

And:

```sql
SELECT
    constraint_schema,
    table_name,
    constraint_name,
    referenced_table_name,
    delete_rule,
    update_rule
FROM information_schema.referential_constraints
WHERE constraint_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
);
```

This is particularly important because AWS DMS doesn't propagate `ON DELETE CASCADE` / `ON UPDATE CASCADE` effects through MySQL CDC in the way you might expect. ([AWS Documentation][2])

---

# 10. Index inventory

```sql
SELECT
    table_schema,
    table_name,
    COUNT(*) AS index_count
FROM information_schema.statistics
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
GROUP BY table_schema, table_name
ORDER BY index_count DESC
LIMIT 100;
```

Find very wide indexes:

```sql
SELECT
    table_schema,
    table_name,
    index_name,
    COUNT(*) AS columns_in_index
FROM information_schema.statistics
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
GROUP BY
    table_schema,
    table_name,
    index_name
ORDER BY columns_in_index DESC
LIMIT 100;
```

---

# 11. AUTO_INCREMENT headroom

This is important before migration because DMS does **not automatically migrate AUTO_INCREMENT attributes to the target column**. ([AWS Documentation][2])

Find auto-increment columns:

```sql
SELECT
    table_schema,
    table_name,
    column_name,
    column_type,
    column_key,
    extra
FROM information_schema.columns
WHERE extra LIKE '%auto_increment%'
AND table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
);
```

This needs to be part of your migration validation.

---

# 12. Current connections

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

Better:

```sql
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
    'Threads_connected',
    'Threads_running',
    'Max_used_connections',
    'Connections',
    'Threads_created',
    'Threads_cached'
);
```

---

# 13. Connection rate

```sql
SHOW GLOBAL STATUS LIKE 'Connections';
SHOW GLOBAL STATUS LIKE 'Threads_created';
SHOW GLOBAL STATUS LIKE 'Threads_cached';
```

Take this twice, for example 60 seconds apart.

```text
Connections(t2) - Connections(t1)
----------------------------------
              60
```

gives approximately:

```text
new connections / second
```

---

# 14. Queries per second

MySQL doesn't directly expose "QPS" as a single perfect metric, so calculate it from status counters.

Run:

```sql
SHOW GLOBAL STATUS LIKE 'Queries';
```

Then wait 60 seconds:

```sql
SHOW GLOBAL STATUS LIKE 'Queries';
```

Calculate:

```text
QPS =
(Queries_2 - Queries_1) / 60
```

Also collect:

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
```

---

# 15. Transactions per second

Use:

```sql
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';
```

Take two samples.

Calculate:

```text
TPS =
(
  ΔCom_commit +
  ΔCom_rollback
) / seconds
```

Also:

```sql
SHOW GLOBAL STATUS LIKE 'Handler_commit';
SHOW GLOBAL STATUS LIKE 'Handler_rollback';
```

---

# 16. Read vs write workload

Collect:

```sql
SHOW GLOBAL STATUS LIKE 'Com_select';

SHOW GLOBAL STATUS LIKE 'Com_insert';
SHOW GLOBAL STATUS LIKE 'Com_update';
SHOW GLOBAL STATUS LIKE 'Com_delete';

SHOW GLOBAL STATUS LIKE 'Com_replace';
```

Take samples over 60 seconds.

Then you can calculate:

```text
SELECT/sec
INSERT/sec
UPDATE/sec
DELETE/sec
```

This will be very useful when selecting the AWS EC2 instance.

---

# 17. Binary log generation rate

This is **one of the most important numbers for your DMS migration**.

Check:

```sql
SHOW BINARY LOGS;
```

You'll get:

```text
Log_name
File_size
```

For example:

```text
binlog.000123    524288000
binlog.000124    734003200
```

Take two samples 10 minutes apart.

Calculate:

```text
binlog_generation_rate
=
(bytes_now - bytes_before)
/
time
```

For example:

```text
10 GB generated / hour
```

This tells us the amount of CDC traffic DMS needs to process.

---

# 18. Binary log configuration

Run:

```sql
SHOW VARIABLES WHERE Variable_name IN (
    'log_bin',
    'binlog_format',
    'binlog_row_image',
    'binlog_checksum',
    'binlog_expire_logs_seconds',
    'max_binlog_size',
    'sync_binlog',
    'server_id',
    'log_replica_updates'
);
```

For DMS CDC, AWS recommends `ROW` binlogging and `FULL` row image; the exact Cloud SQL flags need to be checked against the GCP/DMS requirements. ([AWS Documentation][2])

---

# 19. InnoDB configuration

Run:

```sql
SHOW VARIABLES LIKE 'innodb%';
```

But specifically collect:

```sql
SHOW VARIABLES WHERE Variable_name IN (
    'innodb_buffer_pool_size',
    'innodb_buffer_pool_instances',
    'innodb_log_buffer_size',
    'innodb_log_file_size',
    'innodb_log_files_in_group',
    'innodb_flush_log_at_trx_commit',
    'innodb_flush_method',
    'innodb_io_capacity',
    'innodb_io_capacity_max',
    'innodb_read_io_threads',
    'innodb_write_io_threads',
    'innodb_page_cleaners',
    'innodb_open_files',
    'innodb_file_per_table',
    'innodb_flush_neighbors',
    'innodb_adaptive_hash_index',
    'innodb_stats_on_metadata'
);
```

---

# 20. Buffer pool efficiency

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
```

Important variables include:

```text
Innodb_buffer_pool_read_requests
Innodb_buffer_pool_reads
Innodb_buffer_pool_read_ahead
Innodb_buffer_pool_pages_data
Innodb_buffer_pool_pages_free
Innodb_buffer_pool_pages_dirty
```

Calculate buffer pool hit ratio approximately:

```text
1 -
(
  Innodb_buffer_pool_reads /
  Innodb_buffer_pool_read_requests
)
```

A high hit ratio is desirable.

---

# 21. Buffer pool utilization

```sql
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
    'Innodb_buffer_pool_pages_total',
    'Innodb_buffer_pool_pages_data',
    'Innodb_buffer_pool_pages_free',
    'Innodb_buffer_pool_pages_dirty'
);
```

We can determine how much of the buffer pool is actually being used.

---

# 22. InnoDB row operations

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_rows%';
```

And:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock%';
```

Particularly:

```text
Innodb_row_lock_current_waits
Innodb_row_lock_time
Innodb_row_lock_time_avg
Innodb_row_lock_time_max
Innodb_row_lock_waits
```

High values indicate contention.

---

# 23. Deadlocks

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';
```

Also:

```sql
SHOW ENGINE INNODB STATUS\G
```

Save the output.

This is one of the most valuable diagnostics.

---

# 24. Redo/log pressure

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_log%';
```

And:

```sql
SHOW VARIABLES LIKE 'innodb_log%';
```

For MySQL 8.0, redo configuration matters significantly for write-heavy workloads.

---

# 25. Temporary tables

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

You want:

```text
Created_tmp_tables
Created_tmp_disk_tables
Created_tmp_files
```

Calculate:

```text
disk_temp_ratio =
Created_tmp_disk_tables / Created_tmp_tables
```

High disk temporary table activity can indicate query/workload problems.

---

# 26. Table cache

```sql
SHOW GLOBAL STATUS LIKE 'Opened_tables';
SHOW GLOBAL STATUS LIKE 'Open_tables';
SHOW GLOBAL STATUS LIKE 'Table_open_cache_hits';
SHOW GLOBAL STATUS LIKE 'Table_open_cache_misses';
```

And:

```sql
SHOW VARIABLES LIKE 'table_open_cache';
```

---

# 27. Thread cache

```sql
SHOW GLOBAL STATUS LIKE 'Threads_created';
SHOW GLOBAL STATUS LIKE 'Threads_cached';
SHOW VARIABLES LIKE 'thread_cache_size';
```

---

# 28. MySQL memory configuration

Collect:

```sql
SHOW VARIABLES WHERE Variable_name IN (
    'max_connections',
    'sort_buffer_size',
    'join_buffer_size',
    'read_buffer_size',
    'read_rnd_buffer_size',
    'thread_stack',
    'tmp_table_size',
    'max_heap_table_size',
    'table_open_cache',
    'table_definition_cache',
    'performance_schema'
);
```

Don't blindly multiply every per-connection buffer by `max_connections`, but we absolutely need these values when sizing the EC2 host.

---

# 29. Slow queries

If Performance Schema is enabled, get:

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_seconds,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 50;
```

This is **extremely valuable**.

---

# 30. Most frequently executed queries

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(AVG_TIMER_WAIT / 1000000, 2) AS avg_us
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY COUNT_STAR DESC
LIMIT 50;
```

---

# 31. Worst average queries

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_seconds
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 50;
```

---

# 32. Queries scanning huge amounts of data

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT,
    ROUND(
        SUM_ROWS_EXAMINED / NULLIF(COUNT_STAR, 0),
        0
    ) AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 50;
```

---

# 33. Full table scans

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
AND (
    SUM_NO_INDEX_USED > 0
    OR SUM_NO_GOOD_INDEX_USED > 0
)
ORDER BY
    (SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED) DESC
LIMIT 50;
```

---

# 34. Lock waits

```sql
SELECT
    *
FROM performance_schema.data_lock_waits;
```

And:

```sql
SELECT
    *
FROM performance_schema.data_locks;
```

If available on your Cloud SQL version/configuration.

---

# 35. Active processes

```sql
SHOW FULL PROCESSLIST;
```

And:

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND <> 'Sleep'
ORDER BY TIME DESC;
```

---

# 36. MySQL uptime

```sql
SHOW GLOBAL STATUS LIKE 'Uptime';
```

---

# 37. Global status snapshot

I strongly recommend saving the complete output:

```sql
SHOW GLOBAL STATUS;
```

and:

```sql
SHOW GLOBAL VARIABLES;
```

These two snapshots are incredibly valuable.

---

# 38. Replication status on current GCP replicas

Run on each GCP replica:

```sql
SHOW REPLICA STATUS\G
```

For older terminology/version compatibility:

```sql
SHOW SLAVE STATUS\G
```

Important fields:

```text
Seconds_Behind_Source
Replica_IO_Running
Replica_SQL_Running
Last_IO_Error
Last_SQL_Error
Relay_Log_Space
Exec_Source_Log_Pos
Read_Source_Log_Pos
```

---

# 39. Cloud SQL replication information

From GCP:

```bash
gcloud sql instances describe eshopbox-wms-production
```

And:

```bash
gcloud sql instances describe eshopbox-wms-production-replica-2
```

etc.

Also:

```bash
gcloud sql instances list
```

---

# 40. Cloud SQL machine/storage configuration

For each instance:

```bash
gcloud sql instances describe eshopbox-wms-production \
  --format="yaml(name,databaseVersion,region,settings.tier,settings.dataDiskSizeGb,settings.dataDiskType,settings.availabilityType,settings.ipConfiguration,settings.databaseFlags)"
```

Do the same for the replicas.

---

# 41. GCP metrics

SQL queries cannot give you everything.

For the last **7 days**, collect Cloud Monitoring metrics for:

```text
CPU utilization
Memory utilization
Disk utilization
Disk read IOPS
Disk write IOPS
Disk read bytes
Disk write bytes
Network received
Network sent
Connections
Query count
Transaction count
Replication lag
```

And don't just take averages.

Collect:

```text
Average
P50
P95
P99
Maximum
```

The **maximum/P95** values are particularly important for production EC2 sizing.

---

# 42. AWS EC2 sizing

After collecting all this, we can determine something like:

```text
Observed GCP
──────────────────────────
CPU avg       35%
CPU P95       65%
CPU peak      82%

RAM           42 GB used
IOPS          7,500
Throughput    280 MB/s
TPS           1,200
QPS           8,000
Binlog        6 GB/hour
```

Then we can intelligently select:

```text
EC2 instance
+
EBS gp3/io2
+
IOPS
+
throughput
+
RAM
```

rather than blindly using `10 vCPU / 60 GB`.

---

# 43. MySQL 8.0 AWS configuration

Once we have the numbers, we'll create something along these lines.

For example, **not final values yet**:

```ini
[mysqld]

# Identity
server-id=100
bind-address=0.0.0.0

# Storage
datadir=/data/mysql

# InnoDB
default_storage_engine=InnoDB
innodb_buffer_pool_size=45G
innodb_buffer_pool_instances=8

# Redo
innodb_log_buffer_size=256M

# Durability
innodb_flush_log_at_trx_commit=1
sync_binlog=1

# I/O
innodb_flush_method=O_DIRECT
innodb_io_capacity=4000
innodb_io_capacity_max=8000
innodb_read_io_threads=8
innodb_write_io_threads=8

# Binary logging
log_bin=/data/binlog/mysql-bin
binlog_format=ROW
binlog_row_image=FULL
binlog_expire_logs_seconds=604800

# Replication
gtid_mode=ON
enforce_gtid_consistency=ON
log_replica_updates=ON

# Networking
max_connections=...

# Temporary memory
tmp_table_size=...
max_heap_table_size=...

# Tables
table_open_cache=...
table_definition_cache=...

# Slow query
slow_query_log=ON
long_query_time=1
log_queries_not_using_indexes=OFF

# Error
log_error=/var/log/mysql/error.log
```

But **I would not deploy those exact values yet**.

For example, MySQL documentation notes that InnoDB buffer pool sizing must account for OS memory and other MySQL memory consumers; typical guidance is around 50–75% of system memory, but the right value depends on the workload. ([MySQL Developer Zone][3])

MySQL 8.0 also provides `innodb-dedicated-server`, which can automatically configure several InnoDB settings for dedicated database servers, but for a production migration I prefer to understand the workload before locking in explicit values. ([MySQL Developer Zone][4])

---

# 44. AWS MySQL replication configuration

For the AWS primary:

```ini
server-id=100

log_bin=/data/binlog/mysql-bin
binlog_format=ROW
binlog_row_image=FULL

gtid_mode=ON
enforce_gtid_consistency=ON

log_replica_updates=ON

sync_binlog=1
innodb_flush_log_at_trx_commit=1
```

Replica 1:

```ini
server-id=101
read_only=ON
super_read_only=ON

relay_log=/data/relay/mysql-relay
log_replica_updates=ON

gtid_mode=ON
enforce_gtid_consistency=ON
```

Replica 2:

```ini
server-id=102
read_only=ON
super_read_only=ON

relay_log=/data/relay/mysql-relay
log_replica_updates=ON
```

Payments:

```ini
server-id=103
read_only=ON
super_read_only=ON
```

GTID-based replication is useful for your AWS topology because it makes replica provisioning/failover management cleaner, although **AWS DMS itself does not use GTIDs for its source replication process**. ([AWS Documentation][2])

---

# 45. EBS layout

For a serious production MySQL server, I would consider separating:

```text
             EC2
              │
     ┌────────┼───────────┐
     │        │           │
     ▼        ▼           ▼
   Data     Redo        Binlog
   EBS      EBS         EBS
```

For example:

```text
/data/mysql
/data/redo
/data/binlog
```

Whether this is actually beneficial depends on your I/O profile and EBS configuration. A single high-performance gp3/io2 volume can be simpler and excellent; we should decide based on the measured IOPS/throughput.

---

# 46. Backups

Because this is self-managed EC2, **you now own backups**.

I would design:

```text
EC2 MySQL
    │
    ├── Full backup
    │
    ├── Incremental backup
    │
    ├── Binlogs
    │
    ▼
   S3
```

And test:

```text
backup
restore
point-in-time recovery
```

A backup that has never been restored is not a proven backup.

---

# 47. Monitoring

For the AWS production environment, monitor:

```text
CPU
Memory
Disk utilization
Disk IOPS
Disk latency
Disk throughput

MySQL:
Connections
Threads running
QPS
TPS
Slow queries
Buffer pool hit ratio
Dirty pages
Redo pressure
Lock waits
Deadlocks

Replication:
Seconds behind source
SQL thread
IO thread
Relay log
GTID executed
GTID retrieved

DMS:
CDCLatencySource
CDCLatencyTarget
FullLoadProgress
FullLoadThroughput
CDCLatency
Task errors
Replication instance CPU
Replication instance memory
Replication instance storage
```

---

# 48. DMS-specific prerequisite checks

Before starting DMS, verify:

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW VARIABLES LIKE 'binlog_checksum';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

DMS needs CDC-compatible binary logging and appropriate privileges. AWS documents `REPLICATION CLIENT`, `REPLICATION SLAVE`, and `SELECT` privileges for the DMS source user. ([AWS Documentation][2])

Create a dedicated user rather than using your application/root account:

```sql
CREATE USER 'aws_dms'@'%' IDENTIFIED BY 'STRONG_PASSWORD';

GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE
ON *.*
TO 'aws_dms'@'%';

FLUSH PRIVILEGES;
```

The exact grants should be finalized against your Cloud SQL permissions and DMS version/premigration assessment.

---

# 49. DMS target requirements

On the AWS target, DMS needs write permissions.

AWS documents privileges including:

```sql
ALTER
CREATE
DROP
INDEX
INSERT
UPDATE
DELETE
SELECT
CREATE TEMPORARY TABLES
```

and access to `awsdms_control`. ([AWS Documentation][5])

During full load, foreign-key checks need to be handled appropriately, and AWS DMS recommends `local_infile=1` for MySQL targets. ([AWS Documentation][5])

---

# 50. One correction regarding your "MySQL 8.0.4"

You wrote:

> MySQL 8.04

I'm assuming you mean **MySQL 8.0.40**, matching your current Cloud SQL:

```text
MYSQL_8_0_40
```

I recommend the AWS EC2 target initially use the **same MySQL 8.0 minor version, 8.0.40**, rather than introducing a version upgrade during the migration.

So:

```text
GCP
MySQL 8.0.40
       │
       │ DMS
       ▼
AWS
MySQL 8.0.40
```

Then after the migration is stable, you can plan a separate MySQL upgrade.

**Don't combine migration + major database upgrade + performance tuning + application changes into one event.**

---

# The complete assessment package I want from you

If you're going to build this properly, collect these outputs:

```text
01  MySQL version
02  Database sizes
03  Largest 100 tables
04  Storage engines
05  Number of tables
06  Partitioned tables
07  Foreign keys
08  Index inventory
09  AUTO_INCREMENT columns
10  Connections
11  Max connections
12  QPS
13  TPS
14  SELECT/sec
15  INSERT/sec
16  UPDATE/sec
17  DELETE/sec
18  Binlog generation GB/hour
19  Binlog configuration
20  Buffer pool configuration
21  Buffer pool hit ratio
22  InnoDB row locks
23  Deadlocks
24  Redo statistics
25  Temporary tables
26  Table cache
27  Thread cache
28  Slow queries
29  Top queries by execution
30  Top queries by latency
31  Top queries by rows examined
32  Full table scans
33  Processlist
34  GCP CPU
35  GCP RAM
36  GCP IOPS
37  GCP disk throughput
38  GCP network throughput
39  Replica lag
40  Cloud SQL flags
41  Cloud SQL storage configuration
42  DMS prerequisites
```

Then we can turn those numbers into an actual **AWS production design**:

```text
                    ┌─────────────────────────────┐
                    │       GCP Cloud SQL         │
                    │       MySQL 8.0.40          │
                    │          ~800 GB            │
                    └──────────────┬──────────────┘
                                   │
                            VPN / private link
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │          AWS DMS             │
                    │       Full Load + CDC       │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │       AWS EC2 PRIMARY       │
                    │        MySQL 8.0.40         │
                    │                             │
                    │  tuned buffer pool          │
                    │  tuned redo                 │
                    │  ROW binlog                 │
                    │  GTID                       │
                    │  backups → S3               │
                    │  monitoring                  │
                    └──────────────┬──────────────┘
                                   │
                      Native MySQL replication
                                   │
                     ┌─────────────┼─────────────┐
                     ▼             ▼             ▼
                  Replica 1     Replica 2    Payments
```

**This is the right point to pause before creating the EC2 instances.** Give me the output of the assessment queries above first. From those numbers, I can calculate the **AWS EC2 instance family/size, EBS capacity + IOPS + throughput, `innodb_buffer_pool_size`, redo size, connection limits, binlog retention, replication settings, and DMS sizing** rather than guessing.

[1]: https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.GC.html?utm_source=chatgpt.com "Using Google Cloud for MySQL as a source for AWS DMS - AWS Database Migration Service"
[2]: https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.MySQL.html?utm_source=chatgpt.com "Using a MySQL-compatible database as a source for AWS DMS - AWS Database Migration Service"
[3]: https://dev.mysql.com/doc/refman/8.0/en/innodb-init-startup-configuration.html?utm_source=chatgpt.com "MySQL :: MySQL 8.0 Reference Manual :: 17.8.1 InnoDB Startup Configuration"
[4]: https://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html?utm_source=chatgpt.com "MySQL :: MySQL 8.0 Reference Manual :: 3.5 Changes in MySQL 8.0"
[5]: https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.MySQL.html?utm_source=chatgpt.com "Using a MySQL-compatible database as a target for AWS Database Migration Service - AWS Database Migration Service"
