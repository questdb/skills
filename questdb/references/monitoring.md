# Monitoring QuestDB with SQL

QuestDB records its own metrics in tables, so an instance can be monitored
with plain SQL and a Web Console notebook (see `dashboards-mcp.md`), without
Prometheus or Grafana. The Prometheus endpoint (`:9003/metrics`, needs
`metrics.enabled=true`) still exists for external scrapers.

Docs: `https://questdb.com/docs/operations/logging-metrics.md#persisted-metrics`,
`.../operations/monitoring-alerting.md`, `.../query/functions/meta.md`.

## `sys.metrics`

- One row per second (`metrics.persist.interval`), `ts` is the designated
  timestamp, one LONG or DOUBLE column per metric (about 160 columns).
- On by default (`metrics.persist.enabled`), independent of `metrics.enabled`.
  Kept one week (`metrics.persist.ttl`), day partitions converted to Parquet.
- Local node only, not replicated, hidden from `SHOW TABLES` and the schema
  tree. Query it by name, **quoted**: `FROM 'sys.metrics'`.
- Names are the Prometheus names without `questdb_` and `_total`
  (`commits`, `memory_rss`). Labelled metrics use `__`:
  `worker_pool_fiber_queued__shared_query`.
- Counters are cumulative since start and **reset on restart**. Memory gauges
  refresh every 60 s; rows in between repeat the last value.
- Never write to or alter it.

```sql
SELECT "column", type FROM table_columns('sys.metrics') ORDER BY "column";
```

Useful columns: `cpu_questdb_percent`, `cpu_box_percent`, `memory_rss`,
`memory_mem_used` (includes mappings, not RSS), `memory_jvm_total`,
`memory_jvm_free`, `memory_jvm_max`, `memory_tag_MMAP_TABLE_READER`,
`memory_tag_NATIVE_O3`, `commits`, `rollbacks`, `committed_rows`,
`wal_written_rows`, `wal_apply_written_rows`,
`wal_apply_physically_written_rows`, `wal_apply_seq_txn`,
`wal_apply_writer_txn`, `suspended_tables`, `json_queries_completed`,
`pg_wire_queries_completed`, `query_error_counter`, `pg_wire_errors`,
`unhandled_errors`, `json_queries_cache_hits/misses`,
`pg_wire_select_cache_hits/misses`, `http_connections`,
`pg_wire_connections`, `line_tcp_connections`, `line_http_connections`,
`qwp_egress_connections`, `line_tcp_recv_bytes`, `jvm_minor_gc_time`,
`jvm_major_gc_count`, `worker_pool_fiber_queued__<pool>`. Check the list with
`table_columns` before relying on a name.

### Gauges

```sql
SELECT ts, avg(cpu_questdb_percent) AS questdb_cpu_pct, avg(cpu_box_percent) AS host_cpu_pct,
       100.0 * avg(memory_jvm_total - memory_jvm_free) / avg(memory_jvm_max) AS jvm_heap_used_pct
FROM 'sys.metrics' WHERE ts IN '$now-1h..$now' SAMPLE BY 30s;

SELECT ts, avg(memory_rss) / 1073741824.0 AS rss_gib, max(memory_rss) / 1073741824.0 AS rss_peak_gib
FROM 'sys.metrics' WHERE ts IN '$now-6h..$now' SAMPLE BY 1m;
```

### Counters to Rates (restart-safe)

Take `lag()` deltas, drop negative deltas (a restart reset the counter), and
divide by elapsed seconds:

```sql
WITH d AS (
  SELECT ts,
         wal_written_rows - lag(wal_written_rows) OVER (ORDER BY ts) AS wal_d,
         wal_apply_written_rows - lag(wal_apply_written_rows) OVER (ORDER BY ts) AS apply_d,
         (ts - lag(ts) OVER (ORDER BY ts)) / 1000000.0 AS seconds
  FROM 'sys.metrics' WHERE ts IN '$now-6h..$now'
)
SELECT ts,
       sum(CASE WHEN wal_d >= 0 THEN wal_d ELSE 0 END)
         / nullif(sum(CASE WHEN seconds > 0 THEN seconds ELSE 0 END), 0) AS wal_enqueued_rows_sec,
       sum(CASE WHEN apply_d >= 0 THEN apply_d ELSE 0 END)
         / nullif(sum(CASE WHEN seconds > 0 THEN seconds ELSE 0 END), 0) AS wal_applied_rows_sec
FROM d SAMPLE BY 1m;
```

Same shape for queries/s (`json_queries_completed`, `pg_wire_queries_completed`),
errors/min (`query_error_counter`, `unhandled_errors`; drop the division),
ingress MiB/s (`line_tcp_recv_bytes / 1048576.0`), GC ms/min (`jvm_minor_gc_time`).

```sql
-- WAL backlog and suspended tables
SELECT ts,
       max(CASE WHEN wal_apply_seq_txn >= wal_apply_writer_txn
                THEN wal_apply_seq_txn - wal_apply_writer_txn ELSE 0 END) AS pending_txns,
       max(suspended_tables) AS suspended_tables
FROM 'sys.metrics' WHERE ts IN '$now-6h..$now' SAMPLE BY 1m;

-- Restart indicator: a counter went backwards
WITH d AS (SELECT ts, commits, lag(commits) OVER (ORDER BY ts) AS prev FROM 'sys.metrics'
           WHERE ts IN '$now-6h..$now')
SELECT ts, max(CASE WHEN commits < prev THEN 1 ELSE 0 END) AS counter_reset FROM d SAMPLE BY 1m;
```

## `sys.telemetry_wal`

One row per WAL event, by table id. Columns: `created` (designated
timestamp), `event`, `tableId`, `walId`, `seqTxn`, `rowCount`,
`physicalRowCount`, `latency` (ms), `minTimestamp`, `maxTimestamp`.
Event codes: **109** WAL committed, **103** commit-to-apply-start delay,
**105** data applied. A single apply can finish several transactions and
records the same batch duration on each, so percentiles over 105 are
transaction-weighted. Join `tables()` for names and to exclude views:

```sql
-- Apply duration percentiles
SELECT created AS ts, approx_percentile(latency, 0.5) AS p50,
       approx_percentile(latency, 0.95) AS p95, approx_percentile(latency, 0.99) AS p99
FROM 'sys.telemetry_wal'
WHERE created IN '$now-2h..$now' AND event = 105 AND latency >= 0
SAMPLE BY 1m;

-- Commit-to-apply delay per base table (chart with partition_by_column = table_name)
SELECT s.created AS ts, t.table_name, approx_percentile(s.latency, 0.95) AS commit_to_apply_p95_ms
FROM 'sys.telemetry_wal' s JOIN tables() t ON s.tableId = t.id
WHERE s.created IN '$now-2h..$now' AND s.event = 103 AND s.latency >= 0 AND NOT t.matView
SAMPLE BY 1m;

-- Write amplification per table
SELECT s.created AS ts, t.table_name,
       1.0 * sum(CASE WHEN s.physicalRowCount >= 0 THEN s.physicalRowCount ELSE 0 END)
           / nullif(sum(CASE WHEN s.rowCount > 0 THEN s.rowCount ELSE 0 END), 0) AS amplification
FROM 'sys.telemetry_wal' s JOIN tables() t ON s.tableId = t.id
WHERE s.created IN '$now-6h..$now' AND s.event = 105 AND NOT t.matView
SAMPLE BY 5m;
```

"Hot tables only": compute the top 5 table ids by p95 over the last 5 minutes
in a CTE, then join it to the longer series.

## Meta Functions

| Function | Use |
|----------|-----|
| `tables()` | In-memory table list: WAL lag (`wal_txn - table_txn`), `wal_pending_row_count`, `table_suspended`, memory pressure, write amplification, TTL, dedup. Preferred for health checks. |
| `wal_tables()` | Per-table WAL state from disk, with `suspended`, `errorTag`, `errorMessage`, `memoryPressure` |
| `table_partitions('t')` / `SHOW PARTITIONS FROM t` | Partition sizes, row counts, `isParquet`, `deadRows`, `pieceCount` |
| `table_storage()` | Disk usage per table |
| `query_activity()` | Running queries: `query_id`, `username`, `state`, `memory_used`, `memory_limit`, `query`. Cancel with `CANCEL QUERY id` |
| `materialized_views()` / `live_views()` / `views()` | View status, lag, invalidation reason |
| `memory_metrics()`, `reader_pool()`, `writer_pool()` | Memory and pools |
| `storage_policies`, `table_cold_partitions('t')` | Enterprise storage tiers |
| `resource_groups()`, `node_role()` | Enterprise workload groups, replication role |

```sql
-- Tables behind on WAL apply
SELECT table_name, wal_txn - table_txn AS pending_txns, wal_pending_row_count
FROM tables()
WHERE walEnabled AND wal_txn IS NOT NULL AND wal_txn > table_txn
ORDER BY pending_txns DESC;

-- Suspended tables and why
SELECT name, suspended, errorTag, errorMessage FROM wal_tables() WHERE suspended;
-- fix the cause, then: ALTER TABLE t RESUME WAL;
```
