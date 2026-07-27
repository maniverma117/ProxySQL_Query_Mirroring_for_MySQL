# ProxySQL Query Mirroring for MySQL Buffer Pool Warm-up

1. Overview

2. Architecture

3. Directory Structure

4. Docker Compose

5. ProxySQL Configuration

6. Start ProxySQL

7. Configure Backend Servers

8. Configure Monitor User

9. Configure Application User

10. Configure Mirror Rule

11. Validation

12. Monitoring

13. Buffer Pool Validation

14. Rollback

15. Troubleshooting


---

## Directory Structure

```text
proxysql-mirror/

├── docker-compose.yml
├── README.md
└── proxysql/
    ├── proxysql.cnf
    └── data/
```

Everything stays inside one directory.

---

## Docker Compose

We'll make it production ready.

```yaml
version: "3.9"

services:

  proxysql:
    image: proxysql/proxysql:2.7.3
    container_name: proxysql

    restart: unless-stopped

    ports:
      - "6032:6032"
      - "6033:6033"

    volumes:
      - ./proxysql/proxysql.cnf:/etc/proxysql.cnf:ro
      - ./proxysql/data:/var/lib/proxysql

    healthcheck:
      test: ["CMD", "mysql", "-h127.0.0.1", "-P6032", "-uadmin", "-padmin", "-e", "SELECT 1"]
      interval: 30s
      timeout: 5s
      retries: 3

    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "5"
```

The `./proxysql/data` directory will contain the persistent ProxySQL SQLite database. All `SAVE ... TO DISK` commands write to this database, so your configuration survives container restarts.

---

## proxysql.cnf


datadir="/var/lib/proxysql"

admin_variables=
{
    admin_credentials="admin:admin"

    mysql_ifaces="0.0.0.0:6032"

    refresh_interval=2000
}

mysql_variables=
{
    threads=4

    max_connections=4096

    default_query_delay=0

    default_query_timeout=36000000

    interfaces="0.0.0.0:6033"

    monitor_username="monitor"

    monitor_password="monitor"

    ping_interval_server_msec=1000

    ping_timeout_server=200

    monitor_history=600000

    monitor_connect_interval=60000

    monitor_ping_interval=10000

    monitor_read_only_interval=1500

    connect_timeout_server=3000

    monitor_connect_timeout=600

    shun_on_failures=5

    shun_recovery_time_sec=10
}

---



### Add Existing Read Replica

```sql
INSERT INTO mysql_servers
(
    hostgroup_id,
    hostname,
    port,
    comment
)
VALUES
(
    10,
    '10.0.10.25',
    3306,
    'Existing Read Replica'
);
```

### Add New Read Replica

```sql
INSERT INTO mysql_servers
(
    hostgroup_id,
    hostname,
    port,
    comment
)
VALUES
(
    20,
    '10.0.20.15',
    3306,
    'New Read Replica'
);
```

Then

```sql
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

---

## Configure Application User

```sql
INSERT INTO mysql_users
(
    username,
    password,
    default_hostgroup,
    active
)
VALUES
(
    'appuser',
    'password',
    10,
    1
);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

---

## Configure Mirror Rule

Exactly the rule that worked in your POC:

```sql
INSERT INTO mysql_query_rules
(
    rule_id,
    active,
    match_pattern,
    destination_hostgroup,
    mirror_hostgroup,
    apply,
    comment
)
VALUES
(
    1,
    1,
    '^SELECT',
    10,
    20,
    1,
    'Mirror all SELECT queries'
);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## Monitor User

Since your lab showed repeated `Access denied for user 'monitor'`, the README will also include creating the monitor account on **both** MySQL servers.

```sql
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';

GRANT USAGE ON *.* TO 'monitor'@'%';

FLUSH PRIVILEGES;
```

---

## Validation

Exactly the commands we used during the POC:

* `SELECT @@hostname;`
* Enable General Log on both replicas
* Verify mirrored `SELECT` appears on both
* Different-data test (`OLD` vs `NEW`)
* Stop `mysql-new` and verify client still works

---

## Monitoring

Useful ProxySQL queries:

```sql
SELECT * FROM mysql_servers;

SELECT * FROM runtime_mysql_servers;

SELECT * FROM mysql_users;

SELECT * FROM mysql_query_rules;

SELECT * FROM runtime_mysql_query_rules;

SELECT *
FROM stats_mysql_connection_pool;

SELECT *
FROM stats_mysql_query_digest
ORDER BY count_star DESC
LIMIT 20;
```

And MySQL commands:

```sql
SHOW PROCESSLIST;

SHOW REPLICA STATUS;

SHOW GLOBAL STATUS
LIKE 'Innodb_buffer_pool%';

SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

SHOW GLOBAL STATUS WHERE Variable_name LIKE 'Innodb_buffer_pool_pages%';

SHOW GLOBAL STATUS WHERE Variable_name LIKE 'Innodb_buffer_pool_read%';

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_data';

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_free';

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_dirty';

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
```

---

## Rollback

```sql
UPDATE mysql_query_rules
SET active = 0
WHERE rule_id = 1;

LOAD MYSQL QUERY RULES TO RUNTIME;

SAVE MYSQL QUERY RULES TO DISK;
```

or

```sql
DELETE FROM mysql_query_rules
WHERE rule_id = 1;

LOAD MYSQL QUERY RULES TO RUNTIME;

SAVE MYSQL QUERY RULES TO DISK;
```
