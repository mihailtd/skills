---
name: database-postgresql-materialized-views
description: Guides the agent on PostgreSQL materialized views, including when to use them, how to create and index them, refresh strategies, scheduling, observability, and common gotchas.
---

# PostgreSQL Materialized Views

You are an expert on PostgreSQL materialized views. When asked about caching query results, explain the tradeoff between computation-on-read and computation-on-schedule, show how to build materialized views safely, and describe the operational contract that comes with refreshes.

## 1. What a materialized view is

- A materialized view is a physical relation that stores the result set of a query.
- PostgreSQL computes the query when the materialized view is created or refreshed, then reads operate against the stored rows.
- It is not a virtual view; it does not update automatically on every base-table change.
- The freshness of the data is explicit and controlled by the refresh schedule.

### Key distinction

- `VIEW`: computed on read, always current, no storage overhead, full query cost on every access.
- `MATERIALIZED VIEW`: computed on refresh, stored on disk, fast reads, explicit staleness.
- `SUMMARY TABLE`: maintained by application or ETL logic, gives full control over incremental updates but requires more engineering.

## 2. When materialized views are a strong fit

Materialized views are worth evaluating when:
- a query is expensive to compute and is executed repeatedly
- the query joins large tables and aggregates millions of rows
- the result shape is stable and the query pattern is predictable
- the application can tolerate explicit staleness measured in minutes or hours

### Strong-fit workload shapes

- BI dashboards with repeated summary queries
- weekly or daily rollups over large datasets
- pre-aggregations for slow joins across filtered dimensions
- query shapes that change rarely, even if data does

### Practical signal

- baseline query takes tens of seconds or more
- the same expensive query runs dozens or hundreds of times per day
- tuning indexes and buffers no longer gives meaningful improvement
- users complain about timeouts or long dashboard load times

## 3. A motivating example

Baseline query:

```sql
SELECT
    p.category,
    c.region,
    DATE_TRUNC('week', o.order_date) AS week,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.tenant_id = 'acme_corp'
  AND o.order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY p.category, c.region, DATE_TRUNC('week', o.order_date)
ORDER BY total_revenue DESC;
```

- Execution time: 28 seconds.
- Buffers read: 80GB+ across four tables.
- This is an ideal candidate for a materialized view because the shape is stable and the freshness window can be explicit.

## 4. Create a materialized view safely

### Recommended rollout

1. Create the materialized view with `WITH NO DATA`.
2. Define indexes while the object is empty.
3. Populate it with `REFRESH MATERIALIZED VIEW` in a maintenance window.

### Example creation

```sql
CREATE MATERIALIZED VIEW mv_order_revenue_summary AS
SELECT
    o.tenant_id,
    p.category,
    c.region,
    DATE_TRUNC('week', o.order_date) AS week,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY o.tenant_id, p.category, c.region, DATE_TRUNC('week', o.order_date)
WITH NO DATA;
```

### Why `WITH NO DATA`?

- It creates the physical structure without computing rows immediately.
- You can add indexes before the first refresh.
- It avoids an unexpected long transaction during object creation.

## 5. Index the materialized view like a table

- Treat a materialized view as a table and design indexes for its consumers.
- Index the filter columns and the most common grouping/sort patterns.
- The base table indexes no longer matter for reads against the materialized view; the MV indexes matter.

### Example indexes

```sql
CREATE INDEX idx_mv_revenue_tenant_week
ON mv_order_revenue_summary (tenant_id, week);

CREATE INDEX idx_mv_revenue_category
ON mv_order_revenue_summary (category);

CREATE INDEX idx_mv_revenue_region
ON mv_order_revenue_summary (region);
```

### Querying the materialized view

```sql
SELECT category, region, week, order_count, total_revenue
FROM mv_order_revenue_summary
WHERE tenant_id = 'acme_corp'
  AND week >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY total_revenue DESC;
```

- Result: 180 milliseconds on an index scan, no joins, no aggregation.
- You traded a 28-second read for a 4.2-second scheduled refresh.

## 6. Refresh strategies

### Full refresh

```sql
REFRESH MATERIALIZED VIEW mv_order_revenue_summary;
```

- Recomputes the entire MV.
- The materialized view is locked for reads during the refresh.
- Use only when downtime for the object is acceptable or the refresh is short.

### Concurrent refresh

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_revenue_summary;
```

- Builds the new contents without blocking reads.
- Requires a unique index on the MV.
- Slightly more expensive, but preferred for user-facing queries.

### Unique index requirement

```sql
CREATE UNIQUE INDEX idx_mv_revenue_unique
ON mv_order_revenue_summary (tenant_id, category, region, week);
```

- The unique index must cover the row uniqueness of the MV.
- Without it, concurrent refresh is not permitted.

## 7. Refresh contract and freshness

- Materialized views make staleness explicit.
- Define business-level freshness expectations, for example:
  - every 5 minutes for near-real-time dashboards
  - every hour for operational reports
  - daily for executive summaries
- Define operational SLOs:
  - maximum refresh runtime
  - maximum staleness window

### Good practice

- Expose `last_refresh_at` and `staleness` to users.
- Make refresh cadence visible in dashboards or API responses.
- Treat staleness as a feature of the object, not a bug.

## 8. Scheduling refreshes

### One scheduler rule

- Choose one mechanism to own refreshes.
- Avoid multiple independent jobs or application-triggered refreshes that can cause storms.

### Scheduling options

- OS cron + `psql`
- `pg_cron` extension
- Kubernetes `CronJob`
- cloud scheduler / managed task service

### Example cron entry

```cron
0 * * * * postgres psql -d production -c "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_revenue_summary;"
```

### Example `pg_cron`

```sql
SELECT cron.schedule('refresh-revenue-mv', '0 * * * *',
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_revenue_summary$$);
```

### Kubernetes CronJob example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: refresh-mv-revenue
spec:
  schedule: "0 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: refresh
            image: postgres:16
            command:
            - psql
            - -c
            - "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_revenue_summary;"
          restartPolicy: OnFailure
```

### Guard against overlapping refreshes

- If the refresh takes longer than the schedule interval, use an advisory lock.
- Avoid refresh storms and wasted work.

### Advisory lock pattern

```sql
do $$
begin
  if pg_try_advisory_lock(12345) then
    refresh materialized view concurrently mv_order_revenue_summary;
    perform pg_advisory_unlock(12345);
  else
    raise notice 'Refresh already running, skipping';
  end if;
end $$;
```

## 9. Observability and proof points

### Measure the economics

- Baseline slow query: 28 seconds.
- MV read: 180 milliseconds.
- MV refresh: 4.2 seconds.
- If the query runs 40+ times per day, the refresh cost is tiny compared to repeated reads.

### EXPLAIN examples

Baseline query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

MV query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ... FROM mv_order_revenue_summary;
```
```

- Look for index scans instead of sequential scans.
- Confirm much lower `buffers` and `actual time`.

### Track refresh metadata

- PostgreSQL does not track last refresh time automatically.
- Use a small logging table and update it as part of refresh jobs.

```sql
CREATE TABLE mv_refresh_log (
    mv_name text PRIMARY KEY,
    last_refresh_at timestamptz,
    refresh_duration_ms integer
);
```

```sql
do $$
declare
  start_time timestamptz := clock_timestamp();
  end_time timestamptz;
  duration_ms integer;
begin
  refresh materialized view concurrently mv_order_revenue_summary;
  end_time := clock_timestamp();
  duration_ms := extract(epoch from (end_time - start_time)) * 1000;
  insert into mv_refresh_log (mv_name, last_refresh_at, refresh_duration_ms)
  values ('mv_order_revenue_summary', end_time, duration_ms)
  on conflict (mv_name)
  do update set last_refresh_at = excluded.last_refresh_at,
                refresh_duration_ms = excluded.refresh_duration_ms;
end $$;
```

### Surface freshness

```sql
SELECT mv_name,
       last_refresh_at,
       now() - last_refresh_at AS staleness,
       refresh_duration_ms
FROM mv_refresh_log
WHERE mv_name = 'mv_order_revenue_summary';
```

## 10. Common gotchas and fixes

### 10.1 Concurrent refresh fails

**Symptom:** `REFRESH MATERIALIZED VIEW CONCURRENTLY` errors.

**Cause:** missing unique index on the materialized view.

**Fix:** create a unique index on the columns that make each MV row unique.

```sql
CREATE UNIQUE INDEX idx_mv_revenue_unique
ON mv_order_revenue_summary (tenant_id, category, region, week);
```

### 10.2 Refresh jobs overlap

**Symptom:** multiple refresh processes compete for resources.

**Cause:** schedule interval is shorter than refresh runtime.

**Fix:** add advisory locking or lengthen the interval.

### 10.3 Refresh consumes too much IO/CPU

**Symptom:** write performance degrades during refresh.

**Cause:** refresh runs during peak traffic or scans too much data.

**Fix:** move refresh to an off-peak window, simplify the MV query, or partition the refreshed dataset.

### 10.4 Users see stale data and complain

**Symptom:** dashboard values differ from real-time queries.

**Cause:** freshness contract is not visible.

**Fix:** expose `last_refresh_at`, document the refresh schedule, and show staleness in the UI.

### 10.5 MV query slows as data grows

**Symptom:** reads against the MV degrade over time.

**Cause:** missing or misaligned MV indexes.

**Fix:** add indexes that match filter and sort patterns, and re-examine the MV schema.

## 11. Alternatives and when to choose them

### Normal view

- Use when data must be current on every read.
- No storage or refresh management.
- Still pays the full query cost each time.

### Summary table / ETL pipeline

- Use when you need incremental control over updates.
- More engineering cost, but potentially lower refresh work.
- A good choice when application-level semantics or change data capture are already in place.

### Cache layer (Redis, Memcached)

- Use when you need sub-millisecond response times or when your workload is truly at the scale where a dedicated in-memory store is justified.
- For many modern apps, especially on PostgreSQL 18+, Postgres can provide very competitive performance with its AIO subsystem, UUIDv7 locality improvements, and strong internal buffer management.
- External caches are still valid tools, but they should be treated as an optimization for extreme scale or specific workload patterns, not the default architecture.
- Caches are ideal for serving already-computed snapshots, but they add invalidation complexity and a second source of truth.

### TimescaleDB continuous aggregates

- Use when your workload is time-series centric and you want incremental materialization.
- Continuous aggregates handle refresh-like semantics automatically for time-bucketed data.

## 12. Answering style

- Lead with the design choice: repeated heavy computations are a choice, not a badge of honor.
- Use concrete metrics whenever possible.
- Emphasize the contract: fast reads in exchange for explicit refresh cost and staleness.
- Advise `WITH NO DATA` for safe rollout and `CONCURRENTLY` when reads must stay available.
- Mention that indexing the materialized view matters more than base-table indexes for MV reads.
- Frame refresh failures and stale data as operational items, not bugs.
