# postgres-lb-ha-runbook
PostgreSQL 18 HA cluster with repmgr failover, HAProxy RW/RO routing, and PgBouncer connection pooling - 3000 concurrent client load test included.


# 🐘 PostgreSQL 18 HA Cluster
### repmgr + HAProxy + PgBouncer — Load Balancing & Connection Pooling Runbook

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-blue)
![HAProxy](https://img.shields.io/badge/HAProxy-2.x-red)
![PgBouncer](https://img.shields.io/badge/PgBouncer-1.25-green)
![repmgr](https://img.shields.io/badge/repmgr-5.x-orange)

---

## 📋 Project Overview

This project documents a **Production-Grade PostgreSQL 18 High Availability setup** using:

- **repmgr** — automatic failover and replication management
- **HAProxy** — TCP-level RW/RO traffic routing with external health checks
- **PgBouncer** — connection pooling with session/transaction mode support

Applications connect to a single PgBouncer endpoint. HAProxy transparently routes write traffic to the Primary and read traffic across Standbys via roundrobin.

> No Kubernetes, no etcd — pure VM-based HA stack.

---

## 🖥️ Environment

| Hostname | Role |
|----------|------|
| node1 | PostgreSQL 18 Primary |
| node2 | PostgreSQL 18 Standby |
| node3 | PostgreSQL 18 Standby |
| node4 | HAProxy + PgBouncer (Proxy) |

**OS:** Rocky Linux 8
**PostgreSQL:** 18
**HAProxy:** 2.x
**PgBouncer:** 1.25.1
**repmgr:** 5.x
**OS/DB User:** dbadmin
**PGDATA:** /data/pgdata

---

## 🔌 Ports

| Port | Service | Description |
|------|---------|-------------|
| 5432 | PostgreSQL | Direct DB connection |
| 6432 | PgBouncer | Application entry point |
| 5000 | HAProxy | R/W → Primary |
| 5001 | HAProxy | R-only → Standby roundrobin |
| 7000 | HAProxy | Stats Web UI |

---

## 🔀 Traffic Flow

```
Application
    │
    ▼
node4:6432  (PgBouncer)
    ├── lb_test    → node4:5000 (HAProxy) → node1 Primary   [R/W]
    └── lb_test_ro → node4:5001 (HAProxy) → node2, node3    [R-only, roundrobin]
```

---

## 📁 File Locations

```
node4:/home/dbadmin/
├── .pgpass
└── config/
    ├── haproxy/
    │   ├── haproxy.cfg
    │   ├── check_primary.sh
    │   └── check_standby.sh
    └── pgbouncer/
        ├── pgbouncer.ini
        ├── userlist.txt
        ├── test_write.sql
        ├── test_read.sql
        └── run_test.sh

node4:/etc/systemd/system/
├── haproxy-dbadmin.service
└── pgbouncer-dbadmin.service
```

---

## ⚙️ Key Parameters

### HAProxy — haproxy.cfg

```
global
    maxconn     10000
    external-check

defaults
    maxconn     10000
    timeout client  30m
    timeout server  30m

listen postgres_primary       ← port 5000, R/W
    option external-check
    external-check command check_primary.sh
    server node1 node1:5432 check
    server node2 node2:5432 check backup
    server node3 node3:5432 check backup

listen postgres_standby       ← port 5001, R-only
    balance roundrobin
    option external-check
    external-check command check_standby.sh
    server node2 node2:5432 check maxconn 5000
    server node3 node3:5432 check maxconn 5000
    server node1 node1:5432 check backup
```

**Health check logic:**
`check_primary.sh` → `SELECT pg_is_in_recovery()` = `f` → exit 0 (Primary)
`check_standby.sh` → `SELECT pg_is_in_recovery()` = `t` → exit 0 (Standby)

HAProxy external-check passes args: `$1=spoof_addr $2=spoof_port $3=server_addr $4=server_port`
Scripts use `$3` and `$4` for actual host/port.

---

### PgBouncer — pgbouncer.ini

```ini
[databases]
lb_test    = host=127.0.0.1 port=5000 dbname=lb_test pool_size=1000
lb_test_ro = host=127.0.0.1 port=5001 dbname=lb_test pool_size=2000

[pgbouncer]
listen_port               = 6432
auth_type                 = scram-sha-256
pool_mode                 = session
default_pool_size         = 1000
max_client_conn           = 8500
max_db_connections        = 3000
max_user_connections      = 3000
idle_transaction_timeout  = 30
server_idle_timeout       = 60
ignore_startup_parameters = extra_float_digits,options
```

**pool_mode = session:**
Client holds the server connection for the duration of the session.
Required to reach 1000 concurrent sessions per node under SELECT-only load.

**pool_mode = transaction (more efficient for production):**
Connection returned to pool after each transaction.
Under fast SELECT workloads, DB session count stays low — expected behavior, not a bug.

---

## 4️⃣ HA Status (repmgr)

```
 ID | Name  | Role    | Status    | Upstream | repmgrd | Paused?
----+-------+---------+-----------+----------+---------+--------
 1  | node1 | primary | * running |          | running | no
 2  | node2 | standby |   running | node1    | running | no
 3  | node3 | standby |   running | node1    | running | no
```

<img width="945" height="124" alt="image" src="https://github.com/user-attachments/assets/8bd25d23-a6de-4391-b379-31702e2cdab5" />

---

## 5️⃣ Load Balancing Verification

### HAProxy Stats — node4:7000/stats

<img width="1373" height="696" alt="image" src="https://github.com/user-attachments/assets/8c740214-de52-4c30-855f-9b81766af2fe" />

**postgres_primary:** node1 → UP, node2/node3 → DOWN (backup only)
**postgres_standby:** node2 → UP (~1970 sessions), node3 → UP (~1969 sessions)

---

### PgBouncer POOLS

```
database   | pool_mode | sv_idle
-----------+-----------+--------
lb_test    | session   | 1050
lb_test_ro | session   | 1950
```

<img width="1108" height="350" alt="image" src="https://github.com/user-attachments/assets/29de57fa-b8d1-43d5-9c71-c89c8676e69d" />

---

### Session Distribution (pgbench load test)

**node1 — Primary (~1000 sessions)**

<img width="1854" height="296" alt="image" src="https://github.com/user-attachments/assets/776338db-5f6b-444f-b2a0-a556d0824ae2" />

**node2 — Standby (~1000 sessions)**

<img width="1858" height="245" alt="image" src="https://github.com/user-attachments/assets/ff7dc616-a90f-4b26-994e-e1990763a8e0" />


**node3 — Standby (~1000 sessions)**

<img width="1876" height="234" alt="image" src="https://github.com/user-attachments/assets/ebda1e79-f074-4439-828e-c4b6b5f65feb" />

---

## 6️⃣ pgbench Load Test

| Target | DB alias | Clients | Mode |
|--------|----------|---------|------|
| node1 Primary | lb_test | 1600 | R/W |
| node2 + node3 Standby | lb_test_ro | 6000 | R-only |



```bash
#!/bin/bash
set -e
ulimit -n 65536

HOST="127.0.0.1"
PORT=6432
USER="dbadmin"
SCALE=100
DURATION=160
RESULT_DIR="/home/dbadmin/config/pgbouncer/test_results"
TS=$(date +%Y%m%d_%H%M%S)

mkdir -p "$RESULT_DIR"
export PGPASSWORD="2222"

echo "=== Test started: $(date) ==="

/usr/pgsql-16/bin/pgbench \
    -h "$HOST" -p "$PORT" -U "$USER" -d lb_test \
    --client=1600 --jobs=8 --time="$DURATION" --scale="$SCALE" \
    --file=/home/dbadmin/config/pgbouncer/test_write.sql \
    --progress=10 \
    2>&1 | tee "$RESULT_DIR/rw_${TS}.txt" &

PID_RW=$!

/usr/pgsql-16/bin/pgbench \
    -h "$HOST" -p "$PORT" -U "$USER" -d lb_test_ro \
    --client=6000 --jobs=32 --time="$DURATION" --scale="$SCALE" \
    --file=/home/dbadmin/config/pgbouncer/test_read.sql \
    --select-only \
    --progress=10 \
    2>&1 | tee "$RESULT_DIR/ro_${TS}.txt" &

PID_RO=$!

wait $PID_RW
wait $PID_RO

echo ""
echo "=== R/W result (node1 Primary) ==="
grep -E "tps|latency|transactions" "$RESULT_DIR/rw_${TS}.txt" | tail -5

echo ""
echo "=== R-only result (node2+node3 Standby) ==="
grep -E "tps|latency|transactions" "$RESULT_DIR/ro_${TS}.txt" | tail -5

echo ""
echo "=== PgBouncer POOLS ==="
/usr/pgsql-16/bin/psql -h "$HOST" -p "$PORT" -U "$USER" -d pgbouncer -c "SHOW POOLS;"

echo ""
echo "=== Test finished: $(date) ==="


```





```bash
ulimit -n 65536
./run_test.sh
```

> TPS and latency numbers vary depending on server CPU/memory specs.
> The purpose of this test is to verify correct traffic routing and session distribution — not raw performance benchmarking.

---

## 7️⃣ Notes

- `LimitNOFILE=65536` must be set in the PgBouncer systemd service for high client counts.
- `external-check` must be declared in HAProxy `global` block, not just in `listen` blocks.
- HAProxy `maxconn` in `defaults` was the bottleneck for per-node session limits — raised to 10000.
- `pool_mode = transaction` is more efficient for production OLTP. `session` mode was used here to demonstrate stable 1000-session-per-node distribution.
