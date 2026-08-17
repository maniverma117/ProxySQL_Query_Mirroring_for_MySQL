# ProxySQL Query Mirroring for MySQL Buffer Pool Warm-up

## Overview

This guide explains how to deploy ProxySQL using Docker and configure query mirroring so that all production SELECT queries are executed on both the existing Read Replica and the new Read Replica.

- Existing Read Replica (Hostgroup 10) serves the application response.
- New Read Replica (Hostgroup 20) executes mirrored queries only to warm the InnoDB Buffer Pool.
- No application changes are required.

---

# Architecture

                    Applications
                          |
                          |
                     MySQL Protocol
                          |
                    +-------------+
                    |  ProxySQL   |
                    +------+------+
                           |
             +-------------+-------------+
             |                           |
             |                           |
     Existing Read Replica        New Read Replica
        (Hostgroup 10)             (Hostgroup 20)
      Application Response       Mirror Only

---

# Directory Structure

```
proxysql-mirror/
├── docker-compose.yml
├── README.md
└── proxysql/
    ├── proxysql.cnf
    └── data/
```

---

# Docker Compose

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

---

# ProxySQL Configuration

`proxysql/proxysql.cnf`

```ini
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
```

---

# Deploy ProxySQL

Start ProxySQL

```bash
docker compose up -d
```

Verify container

```bash
docker ps
```

View logs

```bash
docker logs -f proxysql
```

---

# Login to ProxySQL

## Admin Interface

Port **6032** is used for administration.

```bash
mysql \
-h127.0.0.1 \
-P6032 \
-uadmin \
-padmin
```

---

## MySQL Client Interface

Port **6033** is used by applications.

```bash
mysql \
-h127.0.0.1 \
-P6033 \
-uappuser \
-pYourPassword
```

At this point the login will fail because no backend servers or users have been configured yet.

Continue with the next steps.

---

# Create Monitor User on Both MySQL Servers

Execute on both replicas.

```sql
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';

GRANT USAGE ON *.* TO 'monitor'@'%';

FLUSH PRIVILEGES;
```

---

# Configure Backend Servers

Add Existing Read Replica

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
    'OLD_REPLICA_IP',
    3306,
    'Existing Read Replica'
);
```

Add New Read Replica

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
    'NEW_REPLICA_IP',
    3306,
    'New Read Replica'
);
```

Apply configuration

```sql
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify

```sql
SELECT *
FROM mysql_servers;
```

Verify runtime

```sql
SELECT *
FROM runtime_mysql_servers;
```

Both servers should be **ONLINE**.

---

# Configure Application User

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
    'AppPassword',
    10,
    1
);

LOAD MYSQL USERS TO RUNTIME;

SAVE MYSQL USERS TO DISK;
```

Verify

```sql
SELECT *
FROM mysql_users;
```

---

# Configure Query Mirroring

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

Verify

```sql
SELECT *
FROM mysql_query_rules;
```

---

# Test ProxySQL

Connect through ProxySQL

```bash
mysql \
-h127.0.0.1 \
-P6033 \
-uappuser \
-pAppPassword
```

Run

```sql
SELECT @@hostname;

SELECT NOW();
```

Expected:

- Client receives response from **Hostgroup 10**.
- Same SELECT is mirrored to **Hostgroup 20**.

---

# Validation

## Enable General Log (Lab Only)

Execute on both MySQL servers

```sql
SET GLOBAL general_log=ON;
SET GLOBAL log_output='FILE';
```

Tail logs

```bash
tail -f /var/lib/mysql/general.log
```

Execute

```sql
SELECT NOW();

SELECT * FROM users;
```

Expected:

- Query appears in **mysql-old**
- Query appears in **mysql-new**
- Client receives only one response

---

# Monitoring

## ProxySQL

```sql
SELECT * FROM mysql_servers;

SELECT * FROM runtime_mysql_servers;

SELECT * FROM mysql_users;

SELECT * FROM mysql_query_rules;

SELECT * FROM runtime_mysql_query_rules;

SELECT * FROM stats_mysql_query_digest;
```

## MySQL

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

# Rollback

Disable mirroring

```sql
UPDATE mysql_query_rules
SET active=0
WHERE rule_id=1;

LOAD MYSQL QUERY RULES TO RUNTIME;

SAVE MYSQL QUERY RULES TO DISK;
```

Or remove the rule completely

```sql
DELETE FROM mysql_query_rules
WHERE rule_id=1;

LOAD MYSQL QUERY RULES TO RUNTIME;

SAVE MYSQL QUERY RULES TO DISK;
```
