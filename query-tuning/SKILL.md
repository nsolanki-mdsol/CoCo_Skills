---
name: query-tuning
description: 
   Topic   Keywords 
   Slow Queries  slow, long running, performance 
   Full Table Scans  table scan, partitions, pruning 
   Memory Spillage  spillage, spill, memory, remote 
   Queue Time  queue, wait, concurrency 
   Warehouse  warehouse, efficiency, size 
   Clustering  cluster, recluster, depth 
   Search Optimization  search, point lookup, equality 
   Materialized Views  MV, aggregation, precompute 
   Query Acceleration  acceleration, QAS, scale factor 
   Anti-Patterns  bad, avoid, SELECT *, cartesian 
---

# Snowflake Query Tuning Skill

Use this skill when the user asks about query performance, slow queries, query optimization, or tuning in Snowflake.

---

## Description

**Purpose:** Diagnose and optimize slow-running Snowflake queries using ACCOUNT_USAGE views and built-in system functions.

**When to Use:**
- Query taking too long
- High credit consumption
- Warehouse performance issues
- Users complaining about slow dashboards/reports
- Pre-production performance validation

**What This Skill Covers:**
- **Diagnostics:** Identify slow queries, full table scans, memory spillage, queue delays
- **Optimization:** Clustering, search optimization, materialized views, query acceleration
- **Analysis:** EXPLAIN plans, partition pruning, warehouse sizing
- **Best Practices:** Anti-patterns to avoid, tuning checklist

**Data Sources:**
- `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` (up to 365 days, ~45 min latency)
- `SNOWFLAKE.INFORMATION_SCHEMA.QUERY_HISTORY` (real-time, last 7 days)
- `SYSTEM$CLUSTERING_INFORMATION`, `SYSTEM$CLUSTERING_DEPTH`
- `SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS`
- `SYSTEM$ESTIMATE_QUERY_ACCELERATION`

**Required Privileges:**
- `IMPORTED PRIVILEGES` on `SNOWFLAKE` database (for ACCOUNT_USAGE)
- `MONITOR` on warehouses (for warehouse metrics)

---

## Key Topics

### Quick Diagnostics
| # | Topic | Keywords |
|---|-------|----------|
| 1 | Find Slow Queries | slow, long running, 24 hours |
| 2 | Full Table Scans | table scan, partitions, 80% |
| 3 | Memory Spillage | spillage, spill, memory, remote storage |
| 4 | Queue Time | queue, wait, overload, concurrency |
| 5 | Expensive by User | user, cost, who, expensive |
| 6 | Warehouse Efficiency | warehouse, efficiency, size |
| 7 | Analyze Query | query_id, specific, analyze |

### Optimization Techniques
| Topic | Keywords |
|-------|----------|
| Clustering | cluster, clustering key, recluster, depth |
| Search Optimization | search, point lookup, equality, substring |
| Materialized Views | MV, materialized, aggregation, precompute |
| Query Acceleration | acceleration, QAS, scale factor |
| Result Caching | cache, cached result |
| EXPLAIN Plan | explain, plan, profile |

### Reference
| Topic | Keywords |
|-------|----------|
| Quick Reference | reference, problem, solution, indicator |
| Tuning Checklist | checklist, review, audit |
| Anti-Patterns | anti-pattern, bad, avoid, mistake |

---

## Quick Diagnostics

### 1. Find Slow Queries (Last 24 Hours)

```sql
SELECT 
    query_id,
    SUBSTR(query_text, 1, 100) AS query_preview,
    user_name,
    warehouse_name,
    warehouse_size,
    ROUND(total_elapsed_time / 1000, 2) AS elapsed_sec,
    ROUND(execution_time / 1000, 2) AS exec_sec,
    ROUND(compilation_time / 1000, 2) AS compile_sec,
    ROUND(queued_overload_time / 1000, 2) AS queue_sec,
    ROUND(bytes_scanned / 1024 / 1024 / 1024, 3) AS gb_scanned,
    rows_produced,
    partitions_scanned,
    partitions_total,
    ROUND(partitions_scanned * 100.0 / NULLIF(partitions_total, 0), 1) AS pct_partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
  AND total_elapsed_time > 30000  -- Over 30 seconds
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

---

### 2. Identify Full Table Scans

```sql
SELECT 
    query_id,
    SUBSTR(query_text, 1, 100) AS query,
    partitions_scanned,
    partitions_total,
    ROUND(partitions_scanned * 100.0 / NULLIF(partitions_total, 0), 1) AS pct_scanned,
    ROUND(bytes_scanned / 1024 / 1024 / 1024, 2) AS gb_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND partitions_total > 100
  AND partitions_scanned >= partitions_total * 0.8  -- 80%+ scan
  AND execution_status = 'SUCCESS'
ORDER BY partitions_scanned DESC
LIMIT 20;
```

**Fix:** Add clustering key on filter columns or ensure WHERE clause uses clustered columns.

---

### 3. Memory Spillage Detection

```sql
SELECT 
    query_id,
    SUBSTR(query_text, 1, 100) AS query,
    warehouse_name,
    warehouse_size,
    ROUND(bytes_spilled_to_local_storage / 1024 / 1024, 2) AS local_spill_mb,
    ROUND(bytes_spilled_to_remote_storage / 1024 / 1024, 2) AS remote_spill_mb,
    ROUND(total_elapsed_time / 1000, 2) AS elapsed_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND (bytes_spilled_to_local_storage > 0 OR bytes_spilled_to_remote_storage > 0)
ORDER BY bytes_spilled_to_remote_storage DESC
LIMIT 20;
```

**Fix:** Use larger warehouse size or optimize query to reduce data volume.

---

### 4. Queue Time Analysis

```sql
SELECT 
    query_id,
    SUBSTR(query_text, 1, 80) AS query,
    warehouse_name,
    ROUND(queued_overload_time / 1000, 2) AS queue_sec,
    ROUND(queued_provisioning_time / 1000, 2) AS provision_sec,
    ROUND(total_elapsed_time / 1000, 2) AS total_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND queued_overload_time > 5000  -- Over 5 seconds queued
ORDER BY queued_overload_time DESC
LIMIT 20;
```

**Fix:** Scale up warehouse, enable auto-scaling, or use multi-cluster warehouse.

---

### 5. Expensive Queries by User

```sql
SELECT 
    user_name,
    COUNT(*) AS query_count,
    ROUND(SUM(total_elapsed_time) / 1000 / 60, 2) AS total_minutes,
    ROUND(AVG(total_elapsed_time) / 1000, 2) AS avg_seconds,
    ROUND(SUM(bytes_scanned) / 1024 / 1024 / 1024, 2) AS total_gb_scanned,
    SUM(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 ELSE 0 END) AS spill_count
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
GROUP BY user_name
ORDER BY total_minutes DESC
LIMIT 20;
```

---

### 6. Warehouse Efficiency

```sql
SELECT 
    warehouse_name,
    warehouse_size,
    COUNT(*) AS query_count,
    ROUND(AVG(total_elapsed_time) / 1000, 2) AS avg_elapsed_sec,
    ROUND(AVG(execution_time) / 1000, 2) AS avg_exec_sec,
    ROUND(AVG(queued_overload_time) / 1000, 2) AS avg_queue_sec,
    ROUND(SUM(bytes_scanned) / 1024 / 1024 / 1024, 2) AS total_gb_scanned,
    SUM(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 ELSE 0 END) AS spill_count
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
  AND warehouse_name IS NOT NULL
GROUP BY warehouse_name, warehouse_size
ORDER BY avg_elapsed_sec DESC;
```

---

### 7. Analyze Specific Query

```sql
-- Replace QUERY_ID with actual query ID
SELECT 
    query_id,
    query_text,
    total_elapsed_time / 1000 AS elapsed_sec,
    compilation_time / 1000 AS compile_sec,
    execution_time / 1000 AS exec_sec,
    queued_overload_time / 1000 AS queue_sec,
    bytes_scanned / 1024 / 1024 / 1024 AS gb_scanned,
    rows_produced,
    partitions_scanned,
    partitions_total,
    bytes_spilled_to_local_storage / 1024 / 1024 AS local_spill_mb,
    bytes_spilled_to_remote_storage / 1024 / 1024 AS remote_spill_mb,
    query_acceleration_bytes_scanned,
    query_acceleration_partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<QUERY_ID>';
```

---

## Optimization Techniques

### Clustering

```sql
-- Check clustering info
SELECT SYSTEM$CLUSTERING_INFORMATION('my_db.my_schema.my_table', '(date_column)');

-- Add clustering key
ALTER TABLE my_table CLUSTER BY (date_column, region);

-- Manually recluster (if needed)
ALTER TABLE my_table RECLUSTER;

-- Check clustering depth (lower is better, 1-2 is ideal)
SELECT SYSTEM$CLUSTERING_DEPTH('my_table');

-- Suspend/resume automatic clustering
ALTER TABLE my_table SUSPEND RECLUSTER;
ALTER TABLE my_table RESUME RECLUSTER;
```

### Search Optimization

```sql
-- Estimate benefit and cost
SELECT SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS('my_db.my_schema.my_table');

-- Enable search optimization (entire table)
ALTER TABLE my_table ADD SEARCH OPTIMIZATION;

-- For specific columns
ALTER TABLE my_table ADD SEARCH OPTIMIZATION ON EQUALITY(column1, column2);
ALTER TABLE my_table ADD SEARCH OPTIMIZATION ON SUBSTRING(text_column);
ALTER TABLE my_table ADD SEARCH OPTIMIZATION ON GEO(geo_column);

-- Check search optimization status
DESCRIBE SEARCH OPTIMIZATION ON my_table;

-- Remove search optimization
ALTER TABLE my_table DROP SEARCH OPTIMIZATION;
```

### Materialized Views

```sql
-- Create materialized view for expensive aggregations
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    DATE(order_date) AS order_day,
    region,
    SUM(amount) AS total_amount,
    COUNT(*) AS order_count
FROM orders
GROUP BY DATE(order_date), region;

-- Check MV refresh status
SHOW MATERIALIZED VIEWS LIKE 'mv_daily_sales';

-- Manually refresh
ALTER MATERIALIZED VIEW mv_daily_sales REFRESH;

-- Suspend/resume refresh
ALTER MATERIALIZED VIEW mv_daily_sales SUSPEND;
ALTER MATERIALIZED VIEW mv_daily_sales RESUME;
```

### Query Acceleration Service

```sql
-- Check if query acceleration helped
SELECT 
    query_id,
    query_acceleration_bytes_scanned,
    query_acceleration_partitions_scanned,
    query_acceleration_upper_limit_scale_factor
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_acceleration_bytes_scanned > 0
  AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP());

-- Enable query acceleration on warehouse
ALTER WAREHOUSE my_wh SET ENABLE_QUERY_ACCELERATION = TRUE;
ALTER WAREHOUSE my_wh SET QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;

-- Estimate benefit for specific query
SELECT SYSTEM$ESTIMATE_QUERY_ACCELERATION('<query_id>');
```

### Result Caching

```sql
-- Check if result cache is being used
SELECT 
    query_id,
    SUBSTR(query_text, 1, 50) AS query,
    CASE WHEN bytes_scanned = 0 AND rows_produced > 0 THEN 'CACHE HIT' ELSE 'CACHE MISS' END AS cache_status
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
ORDER BY start_time DESC;

-- Enable result caching (account level)
ALTER ACCOUNT SET USE_CACHED_RESULT = TRUE;

-- Session level
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

---

## EXPLAIN Plan

```sql
-- Text format (default)
EXPLAIN 
SELECT * FROM my_table WHERE date_col > '2024-01-01';

-- Explicit text format
EXPLAIN USING TEXT 
SELECT * FROM my_table WHERE date_col > '2024-01-01';

-- JSON format (for programmatic analysis)
EXPLAIN USING JSON
SELECT * FROM my_table WHERE date_col > '2024-01-01';

-- Tabular format
EXPLAIN USING TABULAR
SELECT * FROM my_table WHERE date_col > '2024-01-01';
```

---

## Quick Reference

| Problem | Indicator | Solution |
|---------|-----------|----------|
| Full table scan | `pct_partitions_scanned` > 80% | Add clustering key on filter columns |
| Memory pressure | `bytes_spilled_to_remote_storage` > 0 | Use larger warehouse |
| Queue delays | `queued_overload_time` > 0 | Scale up/out warehouse |
| Slow compilation | `compilation_time` > 5000ms | Simplify query, reduce joins |
| No partition pruning | Low `partitions_total` | Review table design |
| Repeated expensive queries | Same query pattern | Use materialized view |
| Point lookups slow | Equality filters on large table | Enable search optimization |
| Large result sets | High `rows_produced` | Add LIMIT, filter earlier |
| Join explosion | Cartesian product | Review join conditions |

---

## Tuning Checklist

- [ ] Check partition pruning (`pct_partitions_scanned` < 20% ideal)
- [ ] Check for spillage (`bytes_spilled_to_*` should be 0)
- [ ] Check queue time (`queued_overload_time` should be 0)
- [ ] Review EXPLAIN plan for expensive operations
- [ ] Consider clustering keys for frequently filtered columns
- [ ] Consider search optimization for point lookups
- [ ] Consider materialized views for repeated aggregations
- [ ] Review warehouse sizing (try larger if spilling)
- [ ] Check for missing filters (WHERE clause)
- [ ] Check for SELECT * (select only needed columns)
- [ ] Check join order and conditions
- [ ] Consider query acceleration service

---

## Common Anti-Patterns

### 1. SELECT *
```sql
-- Bad
SELECT * FROM large_table;

-- Good
SELECT col1, col2, col3 FROM large_table;
```

### 2. Missing Partition Pruning
```sql
-- Bad (no filter on clustered column)
SELECT * FROM sales WHERE product_id = 123;

-- Good (filter on clustered date column)
SELECT * FROM sales WHERE sale_date > '2024-01-01' AND product_id = 123;
```

### 3. Functions on Filter Columns
```sql
-- Bad (prevents pruning)
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- Good
SELECT * FROM orders WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

### 4. Cartesian Joins
```sql
-- Bad (missing join condition)
SELECT * FROM table_a, table_b;

-- Good
SELECT * FROM table_a JOIN table_b ON table_a.id = table_b.a_id;
```

### 5. ORDER BY Without LIMIT
```sql
-- Bad
SELECT * FROM large_table ORDER BY created_at DESC;

-- Good
SELECT * FROM large_table ORDER BY created_at DESC LIMIT 100;
```
