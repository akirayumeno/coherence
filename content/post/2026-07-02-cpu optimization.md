---
title: "CPU Optimization Case Study: From 99.7% to 19.7%"
date: 2026-07-02T11:30:00+09:00
draft: false
type: "post"
categories:
  - Tech Notes
tags:
  - Performance
  - CPU
  - PostgreSQL
  - Java
  - Tuning
  - Docker
author: "coherence"
---

## Conclusion First

Our production CPU utilization dropped from **99.7% to 19.7%** after applying three focused optimizations:

1. Index addition for high-impact access paths
2. Database cache hit-rate tuning
3. Java heap sizing (`Xmx`) adjustment

This post summarizes the technical approach without exposing any internal SQL text, schema details, or confidential business data.

---

## Context

We observed persistent high CPU usage under normal workload conditions. The issue was not caused by one single bottleneck, but by a combination of:

- inefficient query access patterns,
- low effective memory utilization in the database layer,
- and suboptimal JVM memory configuration in an application container.

Because these three factors amplified each other, we solved them as one coordinated optimization package.

---

## Runbook: Where to Execute Each Check

Use this order during an incident or performance review.

- On host OS: check container CPU and memory pressure.
- In DB container: inspect query behavior, plans, and cache metrics.
- In app container: inspect JVM memory usage and heap settings.

### 0) Baseline Capture (before any tuning)

Run these first and save outputs.

```bash
# Host level
docker stats --no-stream

# DB activity snapshot (inside DB container)
docker exec -it <db-container> /bin/bash
psql -U <db-user> -d <db-name> -c "SELECT now();"

# App JVM snapshot (inside app container)
docker exec -it <app-container> /bin/bash
cat /proc/1/status | grep -i '^Vm\|^Threads'
```

Decision signal:

- If one DB container is CPU-hot and query duration is high, start with index path checks.
- If DB hit rate is low or memory settings are tiny, prioritize hit-rate tuning.
- If JVM memory is tight and GC pressure is visible, prioritize Xmx tuning.

---

## 1) Index Addition (Execution Plan Driven)

### 1-1. Find long-running SQL safely

Use metadata views only. Do not copy internal SQL text into external docs.

```sql
SELECT pid,
       now() - query_start AS duration,
       state
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

Decision signal:

- If duration keeps increasing and CPU is high, move to EXPLAIN ANALYZE.

### 1-2. Validate plan shape

Run EXPLAIN ANALYZE on the target query inside your secure environment.

```sql
EXPLAIN ANALYZE
SELECT ...
FROM ...
WHERE ...
ORDER BY ...
LIMIT ...;
```

How to interpret:

- `Seq Scan` on large tables: candidate for index improvement.
- `Rows Removed by Filter` very large: filter index is likely missing.
- Expensive sort node: consider index aligned with ORDER BY.

### 1-3. Add indexes by access pattern

Choose index type based on where cost appears:

- WHERE column hot: single-column index.
- WHERE + WHERE pattern repeated: composite index.
- JOIN key hot: index on joined column.
- ORDER BY hot: composite index with same column order.

```sql
-- Pattern examples (replace with your own secure names)
CREATE INDEX idx_table_filter_col ON schema.table(filter_col);
CREATE INDEX idx_table_join_col   ON schema.table(join_col);
CREATE INDEX idx_table_f_o        ON schema.table(filter_col, order_col);
```

```sql
ANALYZE schema.table;
```

Re-check:

```sql
EXPLAIN ANALYZE
SELECT ...;
```

Decision signal:

- If plan changes from Seq Scan to Index Scan or Index Only Scan and runtime drops, keep.
- If no plan change, drop or redesign index (wrong column order or low selectivity).

---

## 2) Hit Rate Tuning (PostgreSQL Memory Efficiency)

### 2-1. Measure current hit rate

```sql
SELECT
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit)  AS heap_hit,
  round(sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) AS hit_rate
FROM pg_statio_user_tables;
```

Optional per-table view:

```sql
SELECT
  relname,
  heap_blks_read,
  heap_blks_hit,
  round(heap_blks_hit::numeric /
        nullif(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_rate
FROM pg_statio_user_tables
ORDER BY heap_blks_hit DESC
LIMIT 10;
```

Decision guide (practical heuristic):

- hit_rate < 30%: high priority memory tuning.
- 30% to 70%: tune and verify against workload.
- > 70%: likely not the first bottleneck; check plan quality and JVM.

### 2-2. Inspect current memory settings

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
SHOW config_file;
```

### 2-3. Tune in controlled steps

Edit postgresql.conf and increase gradually.

```conf
shared_buffers = 4GB
work_mem = 256MB
effective_cache_size = 11GB
```

Restart DB service according to your environment policy, then measure again.

Decision signal:

- If hit rate rises and CPU drops without query regressions, keep settings.
- If memory pressure appears (swap/OOM risk), reduce and retest.

---

## 3) JVM Heap (Xmx) Adjustment

### 3-1. Check current JVM memory usage

Inside the application container:

```bash
cat /proc/1/status | grep -i '^Vm'
```

Interpretation:

- `VmRSS`: actual resident memory used now.
- `VmPeak`: peak virtual memory.

If VmRSS is close to container memory limit and CPU is high, GC overhead is a likely contributor.

### 3-2. Increase Xmx safely

Example update in startup script:

```bash
sed -i 's/-Xmx512m/-Xmx1024m/g' /path/to/docker-entrypoint.sh
cat /path/to/docker-entrypoint.sh | grep Xmx
```

Restart the container and re-check VmRSS plus CPU trend.

Decision signal:

- If CPU spikes flatten and response time stabilizes, keep new Xmx.
- If memory contention increases at host level, rebalance Xmx and container limits.

---

## Combined Result and Validation Loop

Validation sequence after each change:

1. Capture docker stats snapshot.
2. Capture query plan and runtime.
3. Capture DB hit rate.
4. Capture JVM VmRSS/VmPeak.

We repeated this loop until all three layers were stable.

Final outcome:

- **Before:** CPU 99.7%
- **After:** CPU 19.7%

The key was not one magic parameter. The key was plan quality + cache efficiency + heap sizing, validated step by step.

---

## Rollback Procedure (Index, Xmx, DB Settings)

Use this when a tuning change causes regression (latency up, errors up, CPU unstable, or memory pressure).

### A) Pre-rollback snapshot

```bash
# Host quick check
docker stats --no-stream

# DB quick check
docker exec -it <db-container> /bin/bash
psql -U <db-user> -d <db-name> -c "SELECT now();"

# App quick check
docker exec -it <app-container> /bin/bash
cat /proc/1/status | grep -i '^Vm\|^Threads'
```

### B) Rollback index changes

If newly added indexes made writes slower or plan quality worse, remove only the candidate indexes.

```sql
-- Identify recently added indexes
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE schemaname = '<schema-name>'
ORDER BY tablename, indexname;
```

```sql
-- Rollback examples (replace with your own names)
DROP INDEX IF EXISTS <schema-name>.idx_table_filter_col;
DROP INDEX IF EXISTS <schema-name>.idx_table_join_col;
DROP INDEX IF EXISTS <schema-name>.idx_table_f_o;
```

```sql
ANALYZE;
```

Decision signal:

- If write throughput recovers or plan cost improves, keep rollback.
- If read latency worsens significantly, restore only the proven-good index.

### C) Rollback Xmx change

If host memory pressure rises after Xmx increase, revert Xmx to previous value.

```bash
# Example: revert 1024m back to 512m
sed -i 's/-Xmx1024m/-Xmx512m/g' /path/to/docker-entrypoint.sh
cat /path/to/docker-entrypoint.sh | grep Xmx
```

```bash
# Restart app container
docker restart <app-container>

# Re-check memory
docker exec -it <app-container> /bin/bash
cat /proc/1/status | grep -i '^Vm'
```

Decision signal:

- If OOM risk or memory contention disappears, keep the rollback value.
- If CPU spikes return, select an intermediate heap size and retest.

### D) Rollback DB memory settings

If DB tuning causes memory pressure or instability, restore prior values in postgresql.conf.

```sql
SHOW config_file;
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
```

Restore previously recorded values in the config file, then restart DB according to your operation policy.

```conf
# Example rollback values
shared_buffers = <previous-value>
work_mem = <previous-value>
effective_cache_size = <previous-value>
```

After restart, verify:

```sql
SELECT
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit)  AS heap_hit,
  round(sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) AS hit_rate
FROM pg_statio_user_tables;
```

### E) Post-rollback acceptance check

Rollback is accepted when all are true:

- Error rate returns to normal.
- CPU trend is stable (no sustained spike pattern).
- Memory pressure is controlled (no swap/OOM alerts).
- Critical transaction latency is back within SLO.

---

## What We Intentionally Redacted

For confidentiality reasons, this article does not include:

- internal SQL statements,
- table/schema names,
- customer or business identifiers,
- raw operational datasets.

The optimization logic, however, is fully reproducible in other environments.

---

## Final Takeaway

If you want repeatable CPU optimization, use an evidence-first workflow:

1. Measure baseline,
2. Change one layer,
3. Re-measure,
4. Keep only what improves both CPU and latency.

In this case, the most reliable sequence was index path tuning, then hit-rate tuning, then Xmx tuning.

