# ProxySQL: Add a Backend Host and User

This guide shows how to add one database endpoint and its dedicated ProxySQL login, load the configuration into runtime, save it to disk, and verify the host-to-user mapping.

> **Important:** Run these commands in the ProxySQL Admin interface on port `6032`, not through the application/MySQL frontend port `3306`. Replace all placeholders before running. Do not commit real passwords to this README or source control.

## 1. Open the ProxySQL Admin console

```bash
docker exec -it proxysql bash
mysql -h127.0.0.1 -P6032 -uadmin -padmin
```

Enter the ProxySQL Admin password when prompted. Avoid putting passwords directly in shell commands.

## 2. Find the next available hostgroup ID

List configured backend hosts:

```sql
SELECT hostgroup_id, hostname, port, comment
FROM mysql_servers
ORDER BY hostgroup_id, hostname;
```

List configured users and their default hostgroups:

```sql
SELECT username, default_hostgroup, active
FROM mysql_users
ORDER BY default_hostgroup, username;
```

Choose an unused hostgroup ID. The examples below use `110`; **verify that it is unused first**:

```sql
SELECT hostgroup_id, hostname
FROM mysql_servers
WHERE hostgroup_id = 110;

SELECT username, default_hostgroup
FROM mysql_users
WHERE username = 'YOUR_USERNAME';
```

Both queries should return no rows for a brand-new hostgroup and username. If the username already exists, do not insert it again; see **Existing username** below.

## 3. Add the backend host

Replace `YOUR_RDS_ENDPOINT` and adjust the hostgroup ID and description as needed.

```sql
INSERT INTO mysql_servers
  (hostgroup_id, hostname, port, status, weight,
   max_connections, max_replication_lag, comment)
VALUES
  (110, 'YOUR_RDS_ENDPOINT', 3306, 'ONLINE', 100,
   1000, 30, 'HG110 - Production RDS');

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

Verify the host is present in runtime:

```sql
SELECT hostgroup_id, hostname, port, status, weight, comment
FROM runtime_mysql_servers
WHERE hostgroup_id = 110;
```

## 4. Add the dedicated user

Use a unique ProxySQL username for each hostgroup. Replace the placeholders; keep the password private.

```sql
INSERT INTO mysql_users
  (username, password, default_hostgroup,
   transaction_persistent, active)
VALUES
  ('YOUR_USERNAME', 'YOUR_PASSWORD', 110, 1, 1);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

If your password contains a single quote (`'`), escape it in the SQL literal by doubling it (`''`). Prefer a secure secret-management workflow rather than storing plaintext passwords in scripts.

Verify the runtime user:

```sql
SELECT username, default_hostgroup, transaction_persistent, active
FROM runtime_mysql_users
WHERE username = 'YOUR_USERNAME';
```

Expected mapping: `YOUR_USERNAME` has `default_hostgroup = 110` and `active = 1`.

## 5. Verify host, user, and connection pool

List all runtime hosts with users whose default hostgroup matches:

```sql
SELECT
  s.hostgroup_id,
  s.hostname,
  s.port,
  s.comment AS host_description,
  u.username,
  u.active AS user_active
FROM runtime_mysql_servers AS s
LEFT JOIN runtime_mysql_users AS u
  ON s.hostgroup_id = u.default_hostgroup
ORDER BY s.hostgroup_id, u.username, s.hostname;
```

Check backend connection-pool stats for the new hostgroup:

```sql
SELECT hostgroup, srv_host, srv_port, status,
       ConnUsed, ConnFree, ConnOK, ConnERR
FROM stats_mysql_connection_pool
WHERE hostgroup = 110;
```

`ONLINE` in `runtime_mysql_servers` confirms the configured runtime entry, but does not alone prove the RDS is reachable. Check connection-pool stats and test a login through the ProxySQL frontend from an allowed client.

## 6. Existing username

ProxySQL enforces uniqueness for the frontend username. If `YOUR_USERNAME` already exists, `INSERT` will fail with a unique-constraint error. If you intentionally want to move that user to hostgroup `110`, update it instead:

```sql
UPDATE mysql_users
SET password = 'YOUR_PASSWORD',
    default_hostgroup = 110,
    transaction_persistent = 1,
    active = 1
WHERE username = 'YOUR_USERNAME';

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

**Caution:** changing `default_hostgroup` changes routing for that username everywhere it is used. Do this only when you intend to move the existing user. To keep access to two separate databases isolated, create a different username for each hostgroup.

## 7. Quick checklist

- [ ] Hostgroup ID is unused.
- [ ] Backend hostname and port are correct.
- [ ] Username does not already exist (or is intentionally updated).
- [ ] Host loaded with `LOAD MYSQL SERVERS TO RUNTIME`.
- [ ] User loaded with `LOAD MYSQL USERS TO RUNTIME`.
- [ ] Both saved with their respective `SAVE ... TO DISK` commands.
- [ ] Runtime host/user mapping verified.
- [ ] RDS security group/network access and database-level grants verified.

## Notes

- `mysql_servers` and `mysql_users` are configuration tables; `runtime_mysql_servers` and `runtime_mysql_users` show the loaded runtime state.
- `default_hostgroup` selects a hostgroup, not an individual hostname. If multiple servers are placed in the same hostgroup, ProxySQL may route the user's connections among eligible servers in that hostgroup.
- This sequence is not an atomic transaction. If any statement fails, stop and inspect the configuration before continuing.
- Do not share or commit real database passwords in this file. Use a secret manager or a secure password-sharing channel.
