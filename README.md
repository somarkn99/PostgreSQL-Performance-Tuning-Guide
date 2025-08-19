
# PostgreSQL Tuning Baselines (Practical Examples)

This companion to `postgres_tuning.md` gives **ready-to-use** starting configs for common RAM sizes and workloads. Treat these as **baselines**—measure, then iterate (see the validation checklist at the end).

> Assumptions
> - PostgreSQL 15+
> - Single PostgreSQL instance per VM
> - OS cache available to Postgres (no huge page constraints)
> - Connection pooling via PgBouncer in **transaction** mode
> - Typical web workload (OLTP) unless stated

---

## Quick Rules of Thumb (re-cap)
- `shared_buffers`: **20–25%** of RAM (cap at ~8–16GB unless DB is huge and IO is fast).
- `effective_cache_size`: **~50–75%** of RAM (what the planner can assume is cached in OS + shared_buffers).
- `work_mem`: **per sort/hash operation, per connection**; start small, scale cautiously. Calculate:  
  `work_mem ≈ (RAM_left_for_work) / (concurrent_active_connections × 2)`
- `maintenance_work_mem`: **0.5–2GB** depending on RAM; used by VACUUM/CREATE INDEX.
- `wal_buffers`: `-1` (auto) is fine; if heavy writes, set **64MB–256MB**.
- `max_connections`: Keep **low** (e.g., 100–200) and use PgBouncer.
- Checkpoints: aim for **stable, infrequent** flushes. `checkpoint_timeout 15min`, `max_wal_size` sized for traffic.
- Autovacuum: keep **on**, but tune thresholds and cost limits for your write rate.

---

## Baseline Tables

### 4 GB RAM (small node, OLTP web)
**Use case:** starter prod, small team, moderate traffic, PgBouncer required.

```ini
# postgresql.conf
max_connections = 150        # rely on PgBouncer
shared_buffers = 1GB         # ~25% of RAM
effective_cache_size = 2.5GB # ~65% of RAM
work_mem = 8MB               # keep conservative
maintenance_work_mem = 512MB
wal_buffers = -1             # auto (≈16MB), can raise to 64MB if heavy writes
effective_io_concurrency = 200  # SSD/EBS gp3/gp2
random_page_cost = 1.1       # fast SSD
seq_page_cost = 1.0
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB
autovacuum_vacuum_cost_limit = 3000
autovacuum_vacuum_cost_delay = 2ms
autovacuum_naptime = 10s
```

**When to adjust**
- Frequent `out of memory` on sorts → raise `work_mem` to 16MB but **watch** RSS.
- WAL spikes/thrashing → increase `max_wal_size` to 6–8GB.
- Slow VACUUM on large tables → raise `maintenance_work_mem` to 1GB during maintenance windows.

---

### 8 GB RAM (balanced OLTP)
**Use case:** typical production for Laravel API with read/write mix.

```ini
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 5GB
work_mem = 16MB
maintenance_work_mem = 1GB
wal_buffers = 64MB           # heavier write bursts
effective_io_concurrency = 200
random_page_cost = 1.1
seq_page_cost = 1.0
checkpoint_timeout = 15min
max_wal_size = 8GB
min_wal_size = 2GB
autovacuum_vacuum_cost_limit = 5000
autovacuum_vacuum_cost_delay = 1ms
autovacuum_naptime = 10s
```

**Variants**
- **Read-heavy**: keep `work_mem=16–32MB`, raise `effective_cache_size=6GB` if OS has room.
- **Write-heavy**: `wal_buffers=128MB`, `max_wal_size=12GB`, `checkpoint_timeout=10min`, consider `synchronous_commit=off` **only** if losing a few seconds of data is acceptable.

---

### 16 GB RAM (higher traffic, still single node)
**Use case:** busy API, complex queries, nightly ETL/maintenance.

```ini
max_connections = 250
shared_buffers = 4GB
effective_cache_size = 10GB
work_mem = 32MB              # monitor memory! large sorts = many MBs * concurrency
maintenance_work_mem = 2GB
wal_buffers = 128MB
effective_io_concurrency = 256
random_page_cost = 1.05
seq_page_cost = 1.0
checkpoint_timeout = 15min
max_wal_size = 16GB
min_wal_size = 4GB
autovacuum_vacuum_cost_limit = 8000
autovacuum_vacuum_cost_delay = 1ms
autovacuum_naptime = 10s
```

**Variants**
- **Reporting/analytics windows**: temporarily bump `work_mem` to `64–128MB` (session-level), and `maintenance_work_mem=4GB` for index builds; revert after.
- **Latency sensitive**: keep `synchronous_commit=on` (default); tune queries/indexes before relaxing durability.

---

## Docker Compose Examples

### Option A — Env overrides
```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: electromall_db
      POSTGRES_USER: electromall_admin
      POSTGRES_PASSWORD: supersecret
      # Optional: extra shared_buffers etc. via PGOPTIONS (not all params allowed)
      # PGOPTIONS: "-c work_mem=16MB -c maintenance_work_mem=1GB"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
```

### Option B — Full config file
```yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
```

### Example `postgresql.conf` (8GB baseline)
```ini
include_if_exists = 'local.conf'

listen_addresses = '*'
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 5GB
work_mem = 16MB
maintenance_work_mem = 1GB
wal_buffers = 64MB
checkpoint_timeout = 15min
max_wal_size = 8GB
min_wal_size = 2GB
effective_io_concurrency = 200
random_page_cost = 1.1
seq_page_cost = 1.0

# logging
log_min_duration_statement = 250ms
log_checkpoints = on
log_autovacuum_min_duration = 0
```

> Put one-off host‑specific tweaks in `local.conf` so you can change without rebuilding the image.

---

## Work_mem Sizing Worksheet

1. Determine **true concurrent active queries** (not connections). With PgBouncer, often **10–50** under load.
2. Estimate **operations per query** (sorts/aggregations/joins). Use **2** as a safe average.  
3. Compute:  
   `work_mem = (RAM_for_work) / (concurrency × ops_per_query)`  
   Example (8GB RAM):
   - Reserve: OS + shared_buffers + other = ~3GB  
   - RAM_for_work ≈ 5GB  
   - concurrency = 30, ops = 2 → `work_mem ≈ 5GB / 60 ≈ 85MB` → **This is too high for baseline.**  
   Start **16–32MB**, increase slowly while watching RSS and `OOMKill`/swap.

> Remember: **work_mem applies per operation**. One connection can consume multiple `work_mem` chunks.

---

## Autovacuum Hints
- Tables with steady writes: lower `autovacuum_vacuum_scale_factor` per table to `0.05` (5%) and set `autovacuum_analyze_scale_factor=0.05`.
- Hot tables: increase `autovacuum_vacuum_cost_limit` (e.g., `10000`) and reduce `cost_delay` to `0.5ms`.
- Use `pg_stat_user_tables` to see dead tuples; `VACUUM (VERBOSE, ANALYZE)` in low-traffic windows if needed.

Per-table example:
```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.05,
  autovacuum_vacuum_cost_limit = 8000
);
```

---

## Validation Checklist

- **Memory**: `ps aux | grep postgres` RSS stable; `docker stats` for container; no swap thrash.  
- **Latency**: p95 below SLO; monitor `pg_stat_statements` for slow queries.  
- **WAL/Checkpoints**: steady graph; no frequent checkpoints; `max_wal_size` not exhausted.  
- **Autovacuum**: no bloated tables; dead tuples under control.  
- **IO**: `pg_statio_*` ratios healthy; `iostat -xm 5` without high `await`.  
- **Connections**: PgBouncer pool hit rate good; `psql "show max_connections"` vs active backends sane.

---

## Quick psql Commands

```sql
-- Top queries
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Cache hit ratio
SELECT sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read),0)::float AS cache_hit_ratio
FROM pg_stat_database;

-- Table bloat suspects (rough)
SELECT schemaname, relname, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## Next Steps
1. Pick the baseline matching your RAM.
2. Apply via compose with a mounted `postgresql.conf` (Option B).
3. Run under typical load (wrk/Gatling + real traffic).
4. Inspect dashboards (Grafana, pgExporter) and **iterate**.
