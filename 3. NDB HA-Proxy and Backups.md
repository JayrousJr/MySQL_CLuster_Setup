# NDB Cluster: HAProxy Load Balancing and Backups

Companion to the main setup guide and the testing guide. Placeholders: `<MANAGER_IP>` (management node), `<HAPROXY_IP>` (machine running HAProxy).

Cluster reference:

| Host | IP | Data node ID | SQL node ID |
|---|---|---|---|
| node1 | 172.15.14.161 | 2 | 11 |
| node2 | 172.15.14.162 | 3 | 12 |
| node3 | 172.15.14.163 | 4 | 13 |

---

# Part 1: HAProxy

## 1.1 What it does

The application connects to **one address**. HAProxy checks which SQL nodes are alive and sends each new connection to a healthy one. It replaces the three-host list in Laravel.

```
Laravel ---> HAProxy :3307 ---> node1 / node2 / node3 (mysqld :3306)
                                     |
                                 data nodes (shared data)
```

> HAProxy balances **SQL nodes**, never data nodes. Data nodes are not addressed by applications.

## 1.2 Install

Lab: run it on the manager machine. Production: run it on its own servers, two of them with keepalived, because one HAProxy is a new single point of failure.

```bash
sudo apt update
sudo apt install -y haproxy
```

## 1.3 Health-check user (all three SQL nodes)

HAProxy logs in to each node with an account that has no password and no privileges, to confirm MySQL really answers. Users are local to each SQL node, so run this on **each** node, using HAProxy's IP (names do not work when `skip-name-resolve` is on):

```sql
CREATE USER 'haproxy_check'@'<HAPROXY_IP>';
```

## 1.4 Configure

Append to `/etc/haproxy/haproxy.cfg`, leaving `global` and `defaults` as they are:

```
listen mysql_cluster
    bind *:3307
    mode tcp
    balance leastconn
    option tcpka
    option mysql-check user haproxy_check
    timeout client 1h
    timeout server 1h
    server node1 172.15.14.161:3306 check inter 3s fall 2 rise 2
    server node2 172.15.14.162:3306 check inter 3s fall 2 rise 2
    server node3 172.15.14.163:3306 check inter 3s fall 2 rise 2

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 5s
```

| Line | Purpose |
|---|---|
| `bind *:3307` | Port 3307 avoids clashing with a MySQL already on that machine |
| `mode tcp` | MySQL is not HTTP; bytes are passed through |
| `balance leastconn` | New connections go to the node with the fewest open ones |
| `mysql-check` | Real MySQL handshake, not just a port check |
| `fall 2 rise 2` | 2 failed checks mark a node down, 2 good ones bring it back |
| `timeout ... 1h` | Prevents idle connections from being cut early |
| `stats` | Web page showing node health |

Validate and apply:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

## 1.5 Point Laravel at HAProxy

`.env`:

```
DB_HOSTS=<HAPROXY_IP>
DB_PORT=3307
```

Then `php artisan config:clear`.

## 1.6 Tests

| # | Test | Action | Expected | Result |
|---|---|---|---|---|
| H1 | Balancing | `mysql -h <HAPROXY_IP> -P 3307 -u appuser -p -e "select @@hostname;"` several times | Hostname changes between nodes | |
| H2 | SQL node down | `systemctl stop mysqld-cluster` on node1, repeat H1 | node1 is never returned; stats page shows it red | |
| H3 | Recovery | `systemctl start mysqld-cluster` on node1 | Green again within a few seconds | |
| H4 | Data node down | Stop one data node | App keeps working; no change in HAProxy | |
| H5 | App | Run the status route loop during H2 | A few failures at most, then all requests succeed | |

Stats page: `http://<HAPROXY_IP>:8404`.

## 1.7 Things to know

- **Existing connections do not move.** If a node dies, connections already open to it break and the app must reconnect. New connections go to a healthy node.
- **`mysql-check` proves only that `mysqld` answers.** A SQL node that lost its link to the data nodes can still pass. Stop it with `systemctl stop mysqld-cluster` and HAProxy removes it.
- **Every client appears as HAProxy's IP** in MySQL. `'appuser'@'%'` works; per-client IP grants become useless.
- **The stats page has no password.** Do not expose port 8404 outside your network.

---

# Part 2: Backups

## 2.1 Strategy

| Layer | Tool | Saves | Restores |
|---|---|---|---|
| Data (native) | `ndb_mgm` `START BACKUP` | All NDB tables, consistent across all nodes, taken online | With `ndb_restore` |
| Data (logical) | `mysqldump` | One database as SQL text | With `mysql < file.sql` |
| Machines | Proxmox backup of each container | OS, cluster software, configs, systemd units, node files | Whole container restore |
| Accounts | Saved SQL script | `CREATE USER` and `GRANT` statements | Run on each SQL node |

Rules:

- Use the native backup and dumps for **data**. Use Proxmox backups for **machines**.
- Native backups do not include users, grants, or InnoDB tables.
- Proxmox snapshots of three data nodes are taken at different moments. Restoring **all three** from them does not give a consistent database. Restoring **one** works: the node catches up from the other two when it starts (use `--initial` once if it refuses to start).
- Keep every backup on **different storage** from the Pi. All containers live on one Pi, so a backup stored there is lost with it.
- Replication is not a backup. A `DELETE` or `DROP` reaches every node instantly.

## 2.2 Native backup

Run on the manager:

```bash
ndb_mgm -e "START BACKUP WAIT COMPLETED"
```

`WAIT COMPLETED` returns only when every data node has finished. It prints a backup number (for example `Backup 1`).

The **manager triggers** the backup, but each **data node writes its own part** on its own disk, by default in `/var/lib/mysql-cluster/BACKUP/BACKUP-<n>/`. A backup is complete only if you keep the folder from **all three** data nodes.

### Option A: copy files off afterwards

From a machine that can reach the nodes:

```bash
mkdir -p ~/backups/ndb/BACKUP-1/node2 ~/backups/ndb/BACKUP-1/node3 ~/backups/ndb/BACKUP-1/node4
scp -r root@172.15.14.161:/var/lib/mysql-cluster/BACKUP/BACKUP-1/* ~/backups/ndb/BACKUP-1/node2/
scp -r root@172.15.14.162:/var/lib/mysql-cluster/BACKUP/BACKUP-1/* ~/backups/ndb/BACKUP-1/node3/
scp -r root@172.15.14.163:/var/lib/mysql-cluster/BACKUP/BACKUP-1/* ~/backups/ndb/BACKUP-1/node4/
```

Then copy `~/backups` to another disk or machine.

### Option B: write straight to a network share

Set `BackupDataDir` in `config.ini` under `[ndbd default]`:

```ini
[ndbd default]
BackupDataDir=/mnt/backup/ndb
```

Requirements:

- The share (NFS, for example) must be mounted at `/mnt/backup/ndb` on **every data node container**, and the `mysql` user must be able to write to it.
- Unprivileged LXC containers cannot mount NFS themselves. Mount the share on the **Proxmox host**, then add it to each container as a mount point (container, Resources, Add, Mount Point).
- Changing `config.ini` needs a manager restart with `--initial` and a rolling restart of the data nodes, one at a time.
- If the share is unavailable, the backup fails. Check `START BACKUP` output and the cluster log after each run.

## 2.3 Logical dump

```bash
mysqldump -h <HAPROXY_IP> -P 3307 -u appuser -p --single-transaction --routines --triggers laravel_clusterdb > laravel_clusterdb.sql
```

It restores anywhere, even on a normal MySQL server. For small data it is the simplest backup to restore.

Save accounts separately in `users.sql`, for example:

```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON laravel_clusterdb.* TO 'appuser'@'%';
CREATE USER 'haproxy_check'@'<HAPROXY_IP>';
```

Run it on each SQL node when rebuilding.

## 2.4 Proxmox backups (machines)

1. Add backup storage in Proxmox (Datacenter, Storage): an NFS share, a second disk, or Proxmox Backup Server. Do not use the Pi's own disk.
2. Datacenter, Backup, Add:
   - Selection: the three node containers (and the manager if it is a container)
   - Storage: the one from step 1
   - Mode: `Snapshot` (needs snapshot-capable storage; otherwise `Suspend` or `Stop`)
   - Schedule: for example weekly
   - Retention: for example keep 4
3. Restore a container with the Restore button on a backup. Afterwards start the data node **without** `--initial` first; if it refuses to start, use `--initial` once so it re-copies from the other nodes.

## 2.4.1 What each tool recovers

| Disaster | Recover with |
|---|---|
| One container lost or broken | Proxmox restore, then let the data node catch up |
| Bad `DELETE` or `DROP` | `ndb_restore` or `mysqldump` file |
| All data nodes lost | Native backup from all three nodes |
| Pi lost | Backups stored on other storage, on new hardware |
| Lost users or grants | `users.sql` |

## 2.5 Restore with `ndb_restore`

**Prepare in advance.** `ndb_restore` joins the cluster as an API client, and the config has no free slot for it ("No free node id"). Add one to `config.ini`:

```ini
[api]
NodeId=20
```

This needs the manager restart and rolling data-node restart described above. Do it while nothing is broken.

Restore into an empty cluster (tables not present), one node's folder at a time. `-n` is the data node ID the backup came from, `-b` the backup number, `-m` restores table structure, `-r` restores rows:

```bash
ndb_restore -c <MANAGER_IP>:1186 -n 2 -b 1 -m -r --backup-path=$HOME/backups/ndb/BACKUP-1/node2
ndb_restore -c <MANAGER_IP>:1186 -n 3 -b 1 -r   --backup-path=$HOME/backups/ndb/BACKUP-1/node3
ndb_restore -c <MANAGER_IP>:1186 -n 4 -b 1 -r   --backup-path=$HOME/backups/ndb/BACKUP-1/node4
```

Existing tables with the same name cause errors. Restore into a fresh database, or drop the old tables first.

Logical restore:

```bash
mysql -h <HAPROXY_IP> -P 3307 -u appuser -p laravel_clusterdb < laravel_clusterdb.sql
```

## 2.6 Restore test (do this before you need it)

- [ ] Insert a few test users
- [ ] Take a native backup and a dump
- [ ] Copy them off the cluster
- [ ] Delete the test users
- [ ] Restore from the dump and confirm the users return
- [ ] Restore from the native backup into a fresh database and confirm
- [ ] Restore one container from a Proxmox backup and confirm the data node catches up

## 2.7 Automation

Weekly cron job on the manager (`crontab -e`). Cron does not use your shell PATH, so use full paths:

```bash
0 2 * * 0 /usr/local/mysql/bin/ndb_mgm -e "START BACKUP WAIT COMPLETED" && /usr/bin/mysqldump -h <HAPROXY_IP> -P 3307 -u appuser -p'<password>' --single-transaction laravel_clusterdb > /backups/dump-$(date +\%F).sql
```

Delete old backups so the disk does not fill, and check a restore regularly.

---

## Checklist

- [ ] HAProxy installed, `haproxy_check` user on all SQL nodes
- [ ] Tests H1 to H5 done
- [ ] Native backup taken and copied off the Pi
- [ ] Logical dump and `users.sql` saved
- [ ] Proxmox backup job running to storage other than the Pi
- [ ] `[api] NodeId=20` slot added to `config.ini`
- [ ] Restore test completed
