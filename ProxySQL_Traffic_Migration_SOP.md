# ProxySQL Traffic Migration SOP

## Architecture

```text
                ProxySQL :6033
                       |
                 application traffic
                       |
                      HG30
                  /          \
             OLD 60%        NEW 40%
```

And **mirroring is completely disabled** before we start traffic shifting.
## Phase 0 — Current state

Your existing mirroring setup is:

```text
SELECT
     ↓
HG10 → mysql-old       ← real traffic
  |
  └── mirror → HG20 → mysql-new
```

The first objective is to remove mirroring and establish:

```text
Application
    ↓
ProxySQL
    ↓
HG30
    ↓
mysql-old
```

Then gradually introduce `mysql-new`.

---

# 1. Stop mirroring

Connect to ProxySQL Admin:

```bash
mysql -h127.0.0.1 -P6032 -uadmin -padmin
```

Check the current mirror rule:

```sql
SELECT
    rule_id,
    active,
    match_pattern,
    destination_hostgroup,
    mirror_hostgroup,
    mirror_flagOUT,
    apply,
    comment
FROM mysql_query_rules
ORDER BY rule_id;
```

You previously had:

```text
rule_id = 1
destination_hostgroup = 10
mirror_hostgroup = 20
```

Disable it:

```sql
UPDATE mysql_query_rules
SET active = 0
WHERE rule_id = 1;
```

Load it:

```sql
LOAD MYSQL QUERY RULES TO RUNTIME;
```

Persist it:

```sql
SAVE MYSQL QUERY RULES TO DISK;
```

Verify:

```sql
SELECT
    rule_id,
    active,
    destination_hostgroup,
    mirror_hostgroup,
    comment
FROM runtime_mysql_query_rules;
```

Expected:

```text
Empty set
```

or no active mirroring rule.

---

# 2. Create Hostgroup 30

We use:

```text
HG10 = OLD
HG20 = NEW
HG30 = Production traffic pool
```

HG30 will contain both servers.

Add OLD:

```sql
INSERT INTO mysql_servers
(
    hostgroup_id,
    hostname,
    port,
    status,
    weight,
    max_connections,
    max_replication_lag,
    max_latency_ms,
    comment
)
VALUES
(
    30,
    'mysql-old',
    3306,
    'ONLINE',
    10,
    1000,
    0,
    0,
    'Production traffic - existing read replica'
);
```

Add NEW:

```sql
INSERT INTO mysql_servers
(
    hostgroup_id,
    hostname,
    port,
    status,
    weight,
    max_connections,
    max_replication_lag,
    max_latency_ms,
    comment
)
VALUES
(
    30,
    'mysql-new',
    3306,
    'ONLINE',
    0,
    1000,
    0,
    0,
    'Production traffic - new read replica'
);
```

### Why NEW starts at weight 0

This gives us a safe baseline:

```text
OLD = 100%
NEW = 0%
```

Load the servers:

```sql
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify:

```sql
SELECT
    hostgroup_id,
    hostname,
    port,
    status,
    weight
FROM runtime_mysql_servers
WHERE hostgroup_id = 30;
```

Expected:

```text
HG30
├── mysql-old   ONLINE   10
└── mysql-new   ONLINE    0
```

---

# 3. Configure the application user for HG30

Your application user should have:

```text
default_hostgroup = 30
```

For example:

```sql
SELECT *
FROM mysql_users;

SELECT
    username,
    active,
    default_hostgroup
FROM mysql_users
WHERE username = 'pocuser';
```

If the production application user already exists, update it:

```sql
SELECT *
FROM mysql_users;

UPDATE mysql_users
SET default_hostgroup = 30
WHERE username = 'YOUR_APP_USER';
```

Then:

```sql
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

Verify:

```sql
SELECT
    username,
    active,
    default_hostgroup
FROM runtime_mysql_users
WHERE username = 'YOUR_APP_USER';
```

It must show:

```text
default_hostgroup = 30
```

---

# 4. Do NOT create a SELECT routing rule

This is important.

Do **not** create:

```sql
match_pattern = '^SELECT'
```

for this migration.

We want the user's default hostgroup to determine routing:

```text
Application
     ↓
pocuser
     ↓
default_hostgroup = 30
     ↓
HG30
```

HG30 itself decides OLD vs NEW based on weight.

---

# 5. Test 100% OLD / 0% NEW

Before moving any traffic, confirm that everything still goes to OLD.

Set:

```sql
UPDATE mysql_servers
SET weight = 10
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 0
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify:

```sql
SELECT
    hostgroup_id,
    hostname,
    status,
    weight
FROM runtime_mysql_servers
WHERE hostgroup_id = 30;
```

Expected:

```text
mysql-old   10
mysql-new    0
```

---

# 6. Test traffic

Use a controlled test first:

```bash
for i in $(seq 1 1000); do
  mysql \
    -h127.0.0.1 \
    -P6033 \
    -upocuser \
    -ppoc123 \
    -N \
    -e "SELECT @@hostname;" 2>/dev/null
done | sort | uniq -c


for i in $(seq 1 100); do
    mysql \
      -h127.0.0.1 \
      -P6033 \
      -uYOUR_APP_USER \
      -pYOUR_PASSWORD \
      -N \
      -e "SELECT @@hostname;" 2>/dev/null
done | sort | uniq -c
```

Expected:

```text
100 mysql-old
```

And:

```text
0 mysql-new
```

Also check ProxySQL:

```sql
SELECT
    hostgroup,
    srv_host,
    Queries
FROM stats_mysql_connection_pool
WHERE hostgroup = 30;


SELECT
    hostgroup,
    srv_host,
    ConnOK,
    ConnERR,
    Queries
FROM stats_mysql_connection_pool
WHERE hostgroup = 30;


SELECT
    hostgroup,
    username,
    digest_text,
    count_star,
    ROUND(sum_time / count_star, 2) AS avg_us,
    min_time,
    max_time
FROM stats_mysql_query_digest
WHERE hostgroup IN (10, 20, 30)
  AND username = 'appuser'
ORDER BY digest_text, hostgroup;
```

```
-- 1. Backend health
SELECT
    hostgroup,
    srv_host,
    Queries,
    ConnOK,
    ConnERR,
    Latency_us
FROM stats_mysql_connection_pool
WHERE hostgroup = 30;

-- 2. Current weights
SELECT
    hostgroup_id,
    hostname,
    status,
    weight
FROM runtime_mysql_servers
WHERE hostgroup_id = 30;

-- 3. Application query performance
SELECT
    hostgroup,
    username,
    digest_text,
    count_star,
    ROUND(sum_time / count_star, 2) AS avg_us,
    min_time,
    max_time
FROM stats_mysql_query_digest
WHERE hostgroup = 30
  AND username = 'appuser'
ORDER BY count_star DESC;

```
---

# 7. Move to 95% OLD / 5% NEW

Once 100/0 is healthy:

```sql
UPDATE mysql_servers
SET weight = 95
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 5
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify:

```sql
SELECT
    hostgroup_id,
    hostname,
    status,
    weight
FROM runtime_mysql_servers
WHERE hostgroup_id = 30;
```

Expected:

```text
mysql-old   95
mysql-new     5
```

### Monitor

Run:

```sql
SELECT
    hostgroup,
    srv_host,
    Queries
FROM stats_mysql_connection_pool
WHERE hostgroup = 30;


SELECT
    hostgroup,
    srv_host,
    ConnUsed,
    ConnFree,
    ConnOK,
    ConnERR,
    Queries
FROM stats_mysql_connection_pool
WHERE hostgroup = 30;
```

Also monitor on both MySQL servers:

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

```sql
SHOW GLOBAL STATUS LIKE 'Threads_running';
```

```sql
SHOW GLOBAL STATUS LIKE 'Queries';
```

And application:

```text
Application errors
Application latency
5xx
DB errors
DB latency
Connection errors
Replication lag
CPU
Memory
Disk I/O
```

Only proceed if NEW is healthy.

---

# 8. Move to 90% OLD / 10% NEW

```sql
UPDATE mysql_servers
SET weight = 90
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 10
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Expected:

```text
OLD = 90
NEW = 10
```

Monitor again.

---

# 9. Move to 80% OLD / 20% NEW

```sql
UPDATE mysql_servers
SET weight = 80
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 20
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Expected:

```text
OLD = 80
NEW = 20
```

Monitor again.

---

# 10. Move to 60% OLD / 40% NEW

This is your major migration checkpoint.

```sql
UPDATE mysql_servers
SET weight = 60
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 40
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Expected:

```text
OLD = 60
NEW = 40
```

Monitor thoroughly before going further.

---

# 11. Move to 50% / 50%

Only after 60/40 is stable:

```sql
UPDATE mysql_servers
SET weight = 50
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 50
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Expected:

```text
OLD = 50
NEW = 50
```

---

# Rollback procedure

This is the most important part.

If anything goes wrong at **any stage**, immediately set:

```sql
UPDATE mysql_servers
SET weight = 100
WHERE hostgroup_id = 30
  AND hostname = 'mysql-old';

UPDATE mysql_servers
SET weight = 0
WHERE hostgroup_id = 30
  AND hostname = 'mysql-new';

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify:

```sql
SELECT
    hostgroup_id,
    hostname,
    status,
    weight
FROM runtime_mysql_servers
WHERE hostgroup_id = 30;
```

You want:

```text
mysql-old   ONLINE   100
mysql-new   ONLINE     0
```

Traffic is then effectively:

```text
Application
     ↓
ProxySQL
     ↓
HG30
     ↓
100% OLD
```

No mirroring is involved.

---

# Final rollout sequence

I recommend keeping this exact sequence:

```text
STEP 1
Disable mirroring
        ↓
STEP 2
Create HG30
        ↓
STEP 3
Add OLD + NEW to HG30
        ↓
STEP 4
Application user → HG30
        ↓
STEP 5
100% OLD / 0% NEW
        ↓
STEP 6
Verify application
        ↓
STEP 7
95% OLD / 5% NEW
        ↓
STEP 8
90% OLD / 10% NEW
        ↓
STEP 9
80% OLD / 20% NEW
        ↓
STEP 10
60% OLD / 40% NEW
        ↓
STEP 11
50% OLD / 50% NEW
```

For each stage:

```text
CHANGE WEIGHT
     ↓
LOAD TO RUNTIME
     ↓
SAVE TO DISK
     ↓
MONITOR
     ↓
HEALTHY?
   /     \
 YES      NO
  ↓        ↓
NEXT     ROLLBACK
```

**One important operational point:** don't interpret the weights as an exact SQL-query percentage. They control ProxySQL's backend selection/connection distribution. Your earlier 50-connection test producing **44 OLD / 6 NEW** with weights 9/1 is exactly the kind of approximate distribution we should expect.
