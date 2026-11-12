# NDB Cluster Failover Testing Guide (Laravel App)

Tests how a Laravel application behaves when it connects to **one** SQL node (Mode A) versus **all three** SQL nodes (Mode B), while nodes are stopped and restarted.

---

## READ THIS FIRST: Data node vs SQL node

> **Your application never connects to a data node. It connects to an SQL node.**
> These are two different programs, and both run on each server (node1, node2, node3).

| | **Data node** (`ndbmtd`) | **SQL node** (`mysqld`) |
|---|---|---|
| What it is | Stores the data in RAM, keeps 3 full copies | The MySQL server on port 3306 |
| Who talks to it | Only SQL nodes and the manager | Your Laravel app |
| Node IDs | 2, 3, 4 | 11, 12, 13 |
| Controlled by | Manager (`ndb_mgm -e "2 stop"`) or `systemctl stop ndbmtd` | Only locally: `systemctl stop mysqld-cluster` |
| If it stops | App keeps working (other data nodes have the copies) | **Any app pointed only at it goes down** |

**Consequences to keep in mind during every test:**

1. **The data is never "on" the node your app is connected to.** All three SQL nodes read the same shared data nodes. Nothing is copied from the connected node to the others.
2. **Stopping a SQL node does not lose data.** It only removes one door into the data.
3. **Stopping a data node does not take the app down**, as long as at least one data node remains and the SQL nodes are up.
4. **Single-host mode has a single point of failure at the SQL node, not at the data.**
5. **Databases (schemas) are per SQL node; NDB tables are shared.** Create `laravel_clusterdb` on every SQL node, or that node will show no tables.

```
App ---> SQL node (mysqld :3306) ---> Data nodes (ndbmtd x3, full copies)
          11 / 12 / 13                    2 / 3 / 4
```

---

## 1. Infrastructure

| Host | IP | Data node ID | SQL node ID |
|---|---|---|---|
| Manager | 172.15.14.176 | n/a (manager ID 1) | n/a |
| node1 | 172.15.14.161 | 2 | 11 |
| node2 | 172.15.14.162 | 3 | 12 |
| node3 | 172.15.14.163 | 4 | 13 |

---

## 2. Test setup

### 2.1 Laravel config

In `config/database.php`, inside the `mysql` connection, replace the single `host` with:

```php
'read'   => ['host' => explode(',', env('DB_HOSTS', '172.15.14.161'))],
'write'  => ['host' => explode(',', env('DB_HOSTS', '172.15.14.161'))],
'sticky' => true,
'engine' => 'NDBCLUSTER',
```

`.env` per mode:

```
# Mode A: single node
DB_HOSTS=172.15.14.161

# Mode B: all three nodes
DB_HOSTS=172.15.14.161,172.15.14.162,172.15.14.163
```

Run `php artisan config:clear` after every change.

### 2.2 Prepare every SQL node (once)

On **each** of node1, node2, node3:

```sql
CREATE DATABASE IF NOT EXISTS laravel_clusterdb;
CREATE USER IF NOT EXISTS 'appuser'@'%' IDENTIFIED BY 'ChooseAStrongPass';
GRANT ALL PRIVILEGES ON laravel_clusterdb.* TO 'appuser'@'%';
```

Run `php artisan migrate` against **one** node only. Then verify on each node:

```sql
SHOW STATUS LIKE 'Ndb_number_of_data_nodes';   -- must be 3
SELECT table_name, engine FROM information_schema.tables WHERE table_schema='laravel_clusterdb';
```

All tables must show `ndbcluster`. Any `InnoDB` table is local to one node and will not replicate.

### 2.3 Status route

```php
Route::get('/status', fn () => response()->json([
    'node'  => DB::selectOne('select @@hostname as h')->h,
    'users' => DB::table('users')->count(),
]));
```

### 2.4 Monitoring (keep running in separate terminals)

```bash
# Request loop
while true; do curl -s -m 3 http://localhost:8000/status; echo; sleep 1; done

# Cluster view (on the manager PC)
watch -n 2 'ndb_mgm -e show'
```

---

## 3. Control commands

| Action | Command |
|---|---|
| Stop a data node (via manager) | `ndb_mgm -e "2 stop"` |
| Restart a data node | `ndb_mgm -e "2 restart"` |
| Stop data node on its own machine | `systemctl stop ndbmtd` |
| Start data node | `systemctl start ndbmtd` |
| Stop SQL node | `systemctl stop mysqld-cluster` |
| Start SQL node | `systemctl start mysqld-cluster` |
| Stop manager only | `ndb_mgm -e "1 stop"` |
| Backup | `ndb_mgm -e "START BACKUP"` |

Warnings:

- `ndb_mgm -e shutdown` and `ndb_mgm -e "all stop"` stop the **whole cluster**. Avoid them in tests.
- `ndbmtd.service` uses `Restart=always`, so `ndb_mgm -e "2 stop"` is followed by an automatic restart after about 10 seconds. Use `systemctl stop ndbmtd` to keep a data node down.
- Never use `--initial` when restarting a data node; it wipes its files.
- Change only one node at a time and wait for `ndb_mgm -e show` to be healthy before the next step.

---

## 4. Mode A: single host

Set `DB_HOSTS=172.15.14.161`, then `php artisan config:clear`.

| # | Test | Action | Expected | Result |
|---|---|---|---|---|
| A1 | Baseline | `ndb_mgm -e show`, open `/status` | All 7 nodes connected, `/status` returns node1 | |
| A2 | Shared data | Register a user, then query `users` on node2 and node3 | User is visible on all three SQL nodes | |
| A3 | SQL node down | `systemctl stop mysqld-cluster` on node1 | **App fails** (connection refused). Data nodes stay healthy; node2's SQL still shows the user | |
| A4 | Recovery | `systemctl start mysqld-cluster` on node1 | App recovers on its own, users intact | |
| A5 | Data node down only | `systemctl stop ndbmtd` on node1 | **App keeps working.** Register a user to prove writes work | |
| A6 | Rejoin | `systemctl start ndbmtd` on node1, wait for `Nodegroup: 0` | The user created in A5 is visible through node1's SQL | |
| A7 | Manager down | `ndb_mgm -e "1 stop"` | App keeps working; start the manager again afterward | |

---

## 5. Mode B: three hosts

Set `DB_HOSTS=172.15.14.161,172.15.14.162,172.15.14.163`, then `php artisan config:clear`.

| # | Test | Action | Expected | Result |
|---|---|---|---|---|
| B1 | Spread | Reload `/status` many times | Rotates between ServerNode1, 2 and 3 across new connections (long-lived workers may stay on one) | |
| B2 | Shared data | Register a user, query each node | User on all three | |
| B3 | One SQL down | Stop `mysqld-cluster` on node1 | **App keeps working** through node2 or node3 | |
| B4 | Two SQL down | Also stop node2's `mysqld-cluster` | node3 serves everything | |
| B5 | All SQL down | Also stop node3's `mysqld-cluster` | App fails; starting any one restores it | |
| B6 | Data node down | Stop one data node, keep SQL nodes up | App works, writes succeed; after restart, new rows are present | |
| B7 | Manager down | `ndb_mgm -e "1 stop"` | App keeps working; restart the manager | |
| B8 | Failure mid-request | Run 20 registrations while stopping a node | One or a few requests fail, then the rest succeed | |
| B9 | Two data nodes down (optional, risky) | Stop two data nodes | Cluster should survive on one. Back up first and do not use real data | |

Note: a request already running on a node when it dies will fail. The next request opens a new connection and skips the dead host.

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Node shows no tables or no data | The database does not exist on that SQL node | `CREATE DATABASE laravel_clusterdb;` there; NDB tables then appear |
| `Ndb_number_of_data_nodes` is 0 | `mysqld` not connected to the cluster | Check `ndbcluster` and `ndb-connectstring` in `/etc/my.cnf`, restart |
| Tables show as `InnoDB` | Created without the NDB engine | Set `'engine' => 'NDBCLUSTER'`, run `php artisan migrate:fresh` |
| `1142 CREATE command denied` | User has no rights on that database on that node | Create `appuser` and grant on every SQL node |
| App fails in failover test only on node2/3 | `appuser` or database missing there | Repeat section 2.2 on that node |
| Stopped data node comes back by itself | `Restart=always` in the unit | Stop with `systemctl stop ndbmtd` |

Caution: never run `DROP TABLE` or `DROP DATABASE` on an NDB table from one node expecting a local effect. It removes the table from the whole cluster. Check the engine first.

---

## 7. Rules for a clean test

- Take `ndb_mgm -e "START BACKUP"` before the optional tests.
- Wait for a healthy `ndb_mgm -e show` between steps.
- Record the result of each test in the Result column.
- Restore everything to the baseline state (A1) before switching modes.
