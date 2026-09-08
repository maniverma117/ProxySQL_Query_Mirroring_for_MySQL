Exactly. For the customer, don't ask for 40 metrics. Give them **one small SQL collection script** that they can run on the GCP Cloud SQL primary and send you the output.

I'd collect **current configuration + database size + workload counters + InnoDB + binlog + replication-related information**.

## 1. Give customer this SQL script

They can run it on the **GCP Cloud SQL primary**.

```sql
-- ============================================================
-- MYSQL 8.0 PRODUCTION MIGRATION ASSESSMENT
-- Run on GCP Cloud SQL PRIMARY
-- ============================================================

-- 1. MySQL / Server information
SELECT
    VERSION() AS mysql_version,
    @@hostname AS hostname,
    @@server_uuid AS server_uuid,
    @@server_id AS server_id,
    @@port AS port,
    @@default_storage_engine AS default_engine,
    @@character_set_server AS character_set,
    @@collation_server AS collation,
    @@time_zone AS timezone;


-- 2. Database size
SELECT
    ROUND(SUM(data_length) / 1024 / 1024 / 1024, 2) AS data_gb,
    ROUND(SUM(index_length) / 1024 / 1024 / 1024, 2) AS index_gb,
    ROUND(
        SUM(data_length + index_length)
        / 1024 / 1024 / 1024,
        2
    ) AS total_database_gb,
    COUNT(DISTINCT table_schema) AS database_count,
    COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
);


-- 3. Database-wise size
SELECT
    table_schema AS database_name,
    COUNT(*) AS tables,
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


-- 4. Largest tables
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
LIMIT 30;


-- 5. Storage engines
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
GROUP BY engine;


-- 6. Partitioned tables
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


-- 7. Current connections
SHOW GLOBAL STATUS LIKE 'Threads_connected';

SHOW GLOBAL STATUS LIKE 'Threads_running';

SHOW GLOBAL STATUS LIKE 'Max_used_connections';

SHOW VARIABLES LIKE 'max_connections';


-- 8. Query / transaction counters
SHOW GLOBAL STATUS LIKE 'Queries';

SHOW GLOBAL STATUS LIKE 'Questions';

SHOW GLOBAL STATUS LIKE 'Com_commit';

SHOW GLOBAL STATUS LIKE 'Com_rollback';


-- 9. Read / write counters
SHOW GLOBAL STATUS LIKE 'Com_select';

SHOW GLOBAL STATUS LIKE 'Com_insert';

SHOW GLOBAL STATUS LIKE 'Com_update';

SHOW GLOBAL STATUS LIKE 'Com_delete';

SHOW GLOBAL STATUS LIKE 'Com_replace';


-- 10. Binary log configuration
SHOW VARIABLES WHERE Variable_name IN (
    'log_bin',
    'binlog_format',
    'binlog_row_image',
    'binlog_checksum',
    'binlog_expire_logs_seconds',
    'max_binlog_size'
);


-- 11. Binary logs
SHOW BINARY LOGS;


-- 12. InnoDB configuration
SHOW VARIABLES WHERE Variable_name IN (
    'innodb_buffer_pool_size',
    'innodb_buffer_pool_instances',
    'innodb_log_buffer_size',
    'innodb_redo_log_capacity',
    'innodb_flush_log_at_trx_commit',
    'innodb_flush_method',
    'innodb_io_capacity',
    'innodb_io_capacity_max',
    'innodb_read_io_threads',
    'innodb_write_io_threads',
    'innodb_file_per_table',
    'innodb_flush_neighbors'
);


-- 13. Buffer pool status
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';


-- 14. InnoDB locks / deadlocks
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock%';

SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';


-- 15. Temporary tables
SHOW GLOBAL STATUS LIKE 'Created_tmp%';


-- 16. Table cache
SHOW GLOBAL STATUS LIKE 'Open_tables';

SHOW GLOBAL STATUS LIKE 'Opened_tables';

SHOW VARIABLES LIKE 'table_open_cache';

SHOW VARIABLES LIKE 'table_definition_cache';


-- 17. Thread cache
SHOW GLOBAL STATUS LIKE 'Threads_created';

SHOW VARIABLES LIKE 'thread_cache_size';


-- 18. Important memory settings
SHOW VARIABLES WHERE Variable_name IN (
    'max_connections',
    'sort_buffer_size',
    'join_buffer_size',
    'read_buffer_size',
    'read_rnd_buffer_size',
    'tmp_table_size',
    'max_heap_table_size',
    'thread_stack'
);


-- 19. Uptime
SHOW GLOBAL STATUS LIKE 'Uptime';


-- 20. Full MySQL configuration
SHOW GLOBAL VARIABLES;
```

---

# 2. One important thing: QPS/TPS needs two samples

The above gives us **counters**, not directly QPS/TPS.

So ask them to run these **two times**, approximately **60 seconds apart**:

```sql
SHOW GLOBAL STATUS
WHERE Variable_name IN (
    'Queries',
    'Questions',
    'Com_commit',
    'Com_rollback',
    'Com_select',
    'Com_insert',
    'Com_update',
    'Com_delete',
    'Threads_connected',
    'Threads_running'
);
```

Example:

```text
Sample 1
11:00:00

Sample 2
11:01:00
```

Then we calculate:

```text
QPS
TPS
SELECT/sec
INSERT/sec
UPDATE/sec
DELETE/sec
connection rate
```

They don't need to calculate anything.

**Just send you both outputs.**

---

# 3. For binlog generation rate

Again, don't make the customer calculate it.

Ask them to run:

```sql
SHOW BINARY LOGS;
```

Then wait **10 minutes** and run:

```sql
SHOW BINARY LOGS;
```

Send both outputs.

We can calculate:

```text
Binlog GB/hour
```

ourselves.

This is particularly important for determining whether DMS can comfortably keep up with CDC during the 800 GB full load.

---

# 4. GCP information — only 3 commands

They can also run these from Cloud Shell:

```bash
gcloud sql instances describe eshopbox-wms-production
```

```bash
gcloud sql instances list
```

And:

```bash
gcloud sql instances describe eshopbox-wms-production-replica-2
```

If they have the other replicas, get:

```bash
gcloud sql instances describe eshopbox-wms-production-read-replica
```

```bash
gcloud sql instances describe eshopbox-wms-production-replica-payments
```

---

# 5. If you want it REALLY simple

Honestly, for the first customer request, I'd send only this.

### Ask customer:

> Please run the following SQL commands on the **GCP Cloud SQL production primary** and send us the outputs. No configuration changes are required.

```sql
SELECT VERSION();

SELECT
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2)
    AS total_database_gb,
    COUNT(*) AS total_tables
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
);

SELECT
    table_schema,
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024 / 1024, 2)
    AS size_gb
FROM information_schema.tables
WHERE table_schema NOT IN (
    'information_schema',
    'mysql',
    'performance_schema',
    'sys'
)
ORDER BY (data_length + index_length) DESC
LIMIT 30;

SHOW GLOBAL STATUS
WHERE Variable_name IN (
    'Threads_connected',
    'Threads_running',
    'Max_used_connections',
    'Queries',
    'Questions',
    'Com_commit',
    'Com_rollback',
    'Com_select',
    'Com_insert',
    'Com_update',
    'Com_delete'
);

SHOW VARIABLES
WHERE Variable_name IN (
    'max_connections',
    'innodb_buffer_pool_size',
    'innodb_log_buffer_size',
    'innodb_redo_log_capacity',
    'innodb_flush_log_at_trx_commit',
    'innodb_io_capacity',
    'innodb_io_capacity_max',
    'innodb_read_io_threads',
    'innodb_write_io_threads',
    'log_bin',
    'binlog_format',
    'binlog_row_image',
    'binlog_expire_logs_seconds'
);

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

SHOW GLOBAL STATUS LIKE 'Innodb_row_lock%';

SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';

SHOW BINARY LOGS;
```

Then:

```text
Run the GLOBAL STATUS query twice,
with approximately 60 seconds between each run.

Run SHOW BINARY LOGS twice,
with approximately 10 minutes between each run.
```

And finally:

```bash
gcloud sql instances list
```

---

## What we can determine from this

With just those outputs, **I can do the engineering work**:

```text
Customer provides
       │
       ▼
┌──────────────────────────────┐
│ DB size          800 GB      │
│ Tables           ???         │
│ Largest tables   ???         │
│ QPS              calculated  │
│ TPS              calculated  │
│ Connections      calculated  │
│ Read/write       calculated  │
│ Binlog GB/hour   calculated  │
│ Buffer pool       ???        │
│ Redo              ???        │
│ Locks             ???        │
└──────────────┬───────────────┘
               │
               ▼
       AWS architecture
               │
       ┌───────┴────────┐
       ▼                ▼
    EC2 size         EBS size
    CPU/RAM          IOPS/throughput
       │                │
       └───────┬────────┘
               ▼
         MySQL 8.0.40
        production config
               │
               ▼
             DMS
        Full Load + CDC
```

And then we can produce the **actual `my.cnf` for the AWS primary and replicas**, rather than guessing buffer pool, redo, connection, I/O, and replication settings.

**I would start with the short version above.** Once we have those results, we can ask the customer for only the *next specific missing piece* if something is genuinely required.
