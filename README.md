# PostgreSQL Performance Tuning Guide + Baselines

This document combines **step-by-step tuning methodology** with **ready-to-use baseline configs** for PostgreSQL.  
It helps you calculate memory budgets, adjust for workloads, and provides concrete example configurations for 4GB, 8GB, and 16GB RAM servers.  

---

## 1) Gather the Baseline
- **Total RAM** on the server (or VM/Container host).
- **Actual concurrency**: how many queries are really active at the same time (not just `max_connections`).
- **PgBouncer presence**: if you have PgBouncer (recommended), actual concurrency is much lower than total client connections.
- **Query types**: heavy sorts/joins? periodic index builds? long reports?

### Quick commands
```bash
# RAM and capacity
free -h

# Active connections
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active';"
psql -c "SELECT usename, count(*) FROM pg_stat_activity GROUP BY 1 ORDER BY 2 DESC;"

# Overall DB activity
psql -c "SELECT datname, xact_commit+xact_rollback AS tps_approx FROM pg_stat_database ORDER BY 2 DESC;"

# Temp file usage (sorts, hashes)
psql -c "SELECT datname, temp_files, pg_size_pretty(temp_bytes) FROM pg_stat_database ORDER BY temp_bytes DESC;"
```

---

## 2) Define the Memory Budget
Divide RAM into:
- **25–30%** → `shared_buffers`
- **70–75%** → `effective_cache_size` (advisory for planner, not real allocation)
- **Remaining** → `work_mem` and OS
- **maintenance_work_mem**: for index builds/vacuum

**Golden rule for work_mem:**  
`work_mem × (concurrent sort/hash operations)` must not exceed available memory.  
Work_mem applies **per operation** inside a query, not per connection.

### Estimating concurrency
- Use “maximum expected heavy queries in parallel.”  
- If ~20–30 heavy queries run in parallel, multiply that by work_mem when budgeting.

**Example (8GB RAM):**
- `shared_buffers=2GB`
- `effective_cache_size=6GB`
- Remaining ~ → work_mem budget  
- Start `work_mem=16MB`, monitor, then increase if needed.

---

## 3) Initial Settings (Start Values)
- `shared_buffers`: 25% of RAM (up to ~8GB before advanced configs)
- `effective_cache_size`: 70–75% of RAM
- `work_mem`: 8–32MB (start at 16MB for ≥8GB RAM)
- `maintenance_work_mem`: 256–1024MB

### Estimating maintenance_work_mem
Check largest index size:
```sql
SELECT relname AS index, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes ui
JOIN pg_index i ON ui.indexrelid=i.indexrelid
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 5;
```

Pick a value close to your largest index size, but not excessively high.  
Typical: 512MB–2GB.

---

## 4) Apply Settings in Docker
### (A) Inline with `command:`
```yaml
postgres:
  image: postgres:15
  command:
    - "postgres"
    - "-c"
    - "shared_buffers=2GB"
    - "-c"
    - "effective_cache_size=6GB"
    - "-c"
    - "work_mem=16MB"
    - "-c"
    - "maintenance_work_mem=512MB"
    - "-c"
    - "max_connections=200"
```

### (B) With custom config file
```yaml
postgres:
  image: postgres:15
  volumes:
    - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
  command: ["postgres","-c","config_file=/etc/postgresql/postgresql.conf"]
```

> With PgBouncer, keep `max_connections` lower (pool_size ~20–50).

---

## 5) Monitor → Adjust
### A) Temp file usage
High `temp_files/temp_bytes` → increase `work_mem` gradually.

```sql
SELECT datname, temp_files, pg_size_pretty(temp_bytes)
FROM pg_stat_database
ORDER BY temp_bytes DESC;
```

### B) Cache hit ratio (target ≥99%)
```sql
SELECT
  sum(blks_hit) / nullif(sum(blks_hit + blks_read),0)::numeric AS cache_hit_ratio
FROM pg_stat_database;
```

Low? → increase `shared_buffers` or RAM, add indexes.

### C) Heavy I/O tables
```sql
SELECT relname, heap_blks_read, heap_blks_hit,
       idx_blks_read, idx_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 10;
```

### D) Query plans
Use `EXPLAIN ANALYZE`.  
If you see “external merge” or “disk” sorts → `work_mem` too small.

---

## 6) Extra Tips
- **PgBouncer**: use `transaction` pooling mode to lower real concurrency.
- **Autovacuum**: keep it enabled, tune if heavy write workload.
- **Indexes**: ensure indexes support common WHERE/ORDER BY/JOIN patterns.
- **WAL/Checkpoints**:
  ```conf
  max_wal_size = 2GB
  checkpoint_completion_target = 0.9
  ```
- **Disk/IOPS**: if slow → SSD with higher IOPS.

---

## 7) Decision Checklist
1. RAM = ________
2. shared_buffers ≈25% → ________
3. effective_cache_size ≈70–75% → ________
4. Actual concurrency = ________
5. work_mem initial = 16MB (adjust by monitoring)
6. maintenance_work_mem = 256MB–2GB
7. Monitor:
   - temp_bytes/temp_files = ________
   - cache_hit_ratio = ________ (≥99%)
   - EXPLAIN ANALYZE shows no disk sorts? Yes/No

---

## 8) Baseline Configurations by RAM Size

### Quick Rules of Thumb
- `shared_buffers`: **20–25%** of RAM (cap at ~8–16GB unless DB is huge and IO is fast).
- `effective_cache_size`: **~50–75%** of RAM.
- `work_mem`: per operation, per connection; start small.
- `maintenance_work_mem`: 0.5–2GB typical.
- `wal_buffers`: auto is fine; set manually (64–128MB) if heavy writes.
- `max_connections`: keep low with PgBouncer.
- `checkpoint_timeout`: 15min, `max_wal_size` sized for workload.
- `autovacuum`: keep enabled, tune thresholds.

---

### Example: 4 GB RAM (small node)
```ini
max_connections = 150
shared_buffers = 1GB
effective_cache_size = 2.5GB
work_mem = 8MB
maintenance_work_mem = 512MB
wal_buffers = -1
effective_io_concurrency = 200
random_page_cost = 1.1
seq_page_cost = 1.0
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB
autovacuum_vacuum_cost_limit = 3000
autovacuum_vacuum_cost_delay = 2ms
autovacuum_naptime = 10s
```

---

### Example: 8 GB RAM (balanced prod)
```ini
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 5GB
work_mem = 16MB
maintenance_work_mem = 1GB
wal_buffers = 64MB
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

---

### Example: 16 GB RAM (busy API)
```ini
max_connections = 250
shared_buffers = 4GB
effective_cache_size = 10GB
work_mem = 32MB
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

---

## 9) Work_mem Worksheet
Formula:  
`work_mem = (RAM_for_work) / (concurrency × ops_per_query)`

Example (8GB RAM, concurrency=30, ops=2):  
RAM_for_work=5GB → 5GB/60 ≈ 85MB (too high for baseline).  
Start **16–32MB**, increase carefully.

---

## 10) Autovacuum Hints
- For hot tables, lower `autovacuum_vacuum_scale_factor` to 0.05, raise cost limit, reduce delay.
- Per-table:
```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.05,
  autovacuum_vacuum_cost_limit = 8000
);
```

---

## 11) Monitoring & Validation
- Check memory with `docker stats` or `ps aux | grep postgres`.
- Cache hit ratio ≥99%.
- WAL/checkpoints stable.
- Autovacuum keeps dead tuples low.
- Query plans free of disk sorts.

### Helpful queries
```sql
-- Top queries
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Cache hit ratio
SELECT sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read),0)::float AS cache_hit_ratio
FROM pg_stat_database;

-- Dead tuples
SELECT schemaname, relname, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## 12) Summary
1. Gather baseline metrics.
2. Define memory budget.
3. Apply starting values (shared_buffers, effective_cache_size, work_mem, maintenance_work_mem).
4. Pick baseline configs as templates for your RAM size.
5. Monitor with pg_stat views and dashboards.
6. Iterate and adjust safely.
