# Views, Retention, Deduplication and Storage

Docs: `https://questdb.com/docs/concepts/materialized-views.md`,
`.../concepts/live-views.md`, `.../concepts/views.md`,
`.../concepts/expire-rows.md`, `.../concepts/deduplication.md`,
`.../concepts/ttl.md`, `.../concepts/storage-policy.md`,
`.../concepts/parquet.md`, `.../concepts/cold-storage.md`.

## Which View?

| Need | Use |
|------|-----|
| Reusable query, always computed on read, parameters | **View** (`CREATE VIEW`) |
| Incremental time-bucketed rollups (candles, hourly stats) | **Materialized view** with `SAMPLE BY` |
| A live copy of some rows or columns of one table | **Passthrough materialized view** (no `SAMPLE BY`) |
| Keep only the latest row per key, top-N, or recent rows | Materialized view + `EXPIRE ROWS` |
| Rolling window functions kept up to date per row (moving averages, Bollinger, session VWAP) | **Live view** (beta) |

---

## Regular Views

```sql
CREATE [OR REPLACE] VIEW [IF NOT EXISTS] viewName AS ( query ) [WITH AUDIT] [OWNED BY owner];
ALTER VIEW viewName AS ( query );
COMPILE VIEW viewName;     -- recompile after schema changes
DROP VIEW viewName;
```

Parameters are `DECLARE` variables. They are fixed unless marked
`OVERRIDABLE`:

```sql
CREATE VIEW trades_for AS (
  DECLARE OVERRIDABLE @symbols := ('BTC-USDT', 'ETH-USDT'),
          OVERRIDABLE @window  := '$now-1h..$now'
  SELECT timestamp, symbol, price, amount FROM trades
  WHERE symbol IN @symbols AND timestamp IN @window
);

DECLARE @symbols := ('SOL-USDT',) SELECT * FROM trades_for;   -- one-item list needs a trailing comma
```

`WITH AUDIT` (Enterprise) records each read and its `AUDITED` parameter values
in `sys.view_audit`.

---

## Materialized Views

```sql
CREATE MATERIALIZED VIEW [IF NOT EXISTS] viewName
[WITH BASE baseTable]
[REFRESH (IMMEDIATE | MANUAL | EVERY interval) [DEFERRED]
         [START ts] [TIME ZONE tz]
         [PERIOD (LENGTH len [TIME ZONE tz] [DELAY d])] [PERIOD (SAMPLE BY INTERVAL)]]
AS ( query )
[TIMESTAMP(col)]
[PARTITION BY YEAR|MONTH|WEEK|DAY|HOUR [TTL n unit]]
[EXPIRE ROWS policy [CLEANUP EVERY duration]]
[OWNED BY owner];
```

- The `REFRESH` clause goes **before** `AS`. Default is `IMMEDIATE`
  (incremental on every base commit). `EVERY` minimum is `1m`. `DEFERRED`
  skips the initial full refresh. `PERIOD` needs `SAMPLE BY`.
- Wrap the query in parentheses when any clause follows it.
- `WITH BASE` is required when the query JOINs. Only the base table triggers
  refresh.
- Default partitioning follows `SAMPLE BY`: over 1h gives YEAR, over 1m gives
  MONTH, 1m or less gives DAY.
- Aggregating queries may use `WHERE` and `JOIN`. Not supported: `FILL`,
  `FROM-TO`, `ALIGN TO FIRST OBSERVATION`, `GROUPING SETS`/`ROLLUP`/`CUBE`,
  non-deterministic functions such as `now()`.
- **Since 10.2.0:** the part of the query that reads the base table (including
  subqueries on it) cannot use `LIMIT` or a window function anywhere in a
  column (`row_number() OVER (...) + 1`). Existing views keep refreshing, but
  recreating one needs the query reworked.
- If the base table has DEDUP, non-aggregate columns must be a subset of the
  DEDUP keys.
- Retention: `TTL` on the view, independent of the base table. Storage
  policies are not allowed on materialized views, even on Enterprise.

```sql
-- Candle cascade: trades -> 1s -> 1m -> 1h
CREATE MATERIALIZED VIEW IF NOT EXISTS trades_ohlc_1s AS (
  SELECT timestamp, symbol,
         first(price) AS open, max(price) AS high, min(price) AS low,
         last(price) AS close, sum(amount) AS volume
  FROM trades SAMPLE BY 1s
) PARTITION BY DAY TTL 7 DAYS;

CREATE MATERIALIZED VIEW IF NOT EXISTS trades_ohlc_1m AS (
  SELECT timestamp, symbol,
         first(open) AS open, max(high) AS high, min(low) AS low,
         last(close) AS close, sum(volume) AS volume
  FROM trades_ohlc_1s SAMPLE BY 1m
) PARTITION BY DAY TTL 90 DAYS;

-- Joined rollup
CREATE MATERIALIZED VIEW trades_with_meta WITH BASE trades AS
SELECT t.timestamp, t.symbol, m.description, avg(t.price) AS avg_price
FROM trades t JOIN instruments m ON t.symbol = m.symbol
SAMPLE BY 1h;

-- Daily bars that wait for late data
CREATE MATERIALIZED VIEW trades_daily
REFRESH PERIOD (LENGTH 1d TIME ZONE 'Europe/London' DELAY 2h) AS
SELECT timestamp, symbol, avg(price) AS avg_price FROM trades SAMPLE BY 1d;
```

Operations:

```sql
REFRESH MATERIALIZED VIEW v [FULL | INCREMENTAL | RANGE FROM ts TO ts];   -- async
ALTER MATERIALIZED VIEW v SET REFRESH EVERY 15m;
ALTER MATERIALIZED VIEW v SET TTL 30 DAYS;
ALTER MATERIALIZED VIEW v SET REFRESH LIMIT 1 WEEK;
ALTER MATERIALIZED VIEW v RESUME WAL;
DROP MATERIALIZED VIEW v;                     -- never DROP TABLE a view
SELECT view_name, view_status, invalidation_reason FROM materialized_views();
```

Never `INSERT` into a view; write to its base table. A view becomes invalid
after base `TRUNCATE`/`UPDATE`, dropped columns or partitions, or a base
rename. Invalidation sticks: fix it with `REFRESH ... FULL`.

### Passthrough Views (10.2.0)

A materialized view without `SAMPLE BY` copies base rows one-for-one: a live
subset of one table's columns or rows.

```sql
CREATE MATERIALIZED VIEW trades_btc AS (
  SELECT timestamp, symbol, price, amount FROM trades WHERE symbol = 'BTC-USDT'
);
```

- Allowed: `SELECT *`, column subsets, aliases, row-local expressions, `WHERE`.
- Rejected: `DISTINCT`, `LATEST ON`, `JOIN`, `UNION`, window functions,
  `LIMIT`, `ORDER BY` a non-timestamp column. `REFRESH PERIOD` is rejected.
- Must keep the designated timestamp. Inherits partitioning and symbol indexes.
- Columns are fixed at creation (`SELECT *` is expanded). Columns added to the
  base later never reach the view; recreate it.

### EXPIRE ROWS (10.2.0)

Keeps only some rows of a materialized view. Queries hide expired rows
immediately; a background job reclaims space when the policy allows it.

```sql
EXPIRE ROWS
  { WHEN predicate
  | KEEP LATEST [ON timestampColumn] PARTITION BY col [, ...]
  | KEEP [N] (HIGHEST | LOWEST) ON col [PARTITION BY col [, ...]] }
  [CLEANUP EVERY duration]          -- default 1h

-- Latest reading per sensor
CREATE MATERIALIZED VIEW sensor_latest AS (
  SELECT timestamp, sensor_id, value FROM readings
) EXPIRE ROWS KEEP LATEST ON timestamp PARTITION BY sensor_id;

-- Rolling 7-day mirror
ALTER MATERIALIZED VIEW trades_mirror
  SET EXPIRE ROWS WHEN timestamp < dateadd('d', -7, now()) CLEANUP EVERY 30m;

-- Top 10 trades by size per symbol
... ) EXPIRE ROWS KEEP 10 HIGHEST ON amount PARTITION BY symbol;

ALTER MATERIALIZED VIEW v DROP EXPIRE;
SELECT view_name, expire_enforcement FROM materialized_views();
```

- Only a plain `WHEN` frees disk (`FILTER_AND_RECLAIM`). `KEEP` modes and
  window conditions only hide rows (`FILTER_ONLY`).
- Not allowed on tables. With `TTL`, TTL comes first in the statement and acts first.
- No `--` comments inside the clause. A NULL constant threshold is rejected.
- No materialized or live view can read a view that has an active policy.

---

## Live Views (beta)

A live view stores the incrementally maintained result of a **window-function
query over one WAL table**: one output row per base row, queryable like a
table, updated as rows arrive.

```sql
CREATE LIVE VIEW [IF NOT EXISTS] viewName
FLUSH EVERY duration              -- required, first; 100ms .. 60m
[IN MEMORY duration]              -- >= FLUSH EVERY
[PARTITION BY YEAR|MONTH|WEEK|DAY|HOUR]
START FROM { NOW | BEGINNING | 'timestamp' }
AS { query | ( query ) [OWNED BY owner] };
```

Durations are single tokens: `500ms`, `1s`, `5m`.

```sql
-- Rolling Bollinger inputs, per symbol
CREATE LIVE VIEW trades_bollinger
FLUSH EVERY 1s
START FROM NOW
AS
SELECT timestamp, symbol, price,
       avg(price) OVER w AS sma20,
       sqrt(greatest(avg(price * price) OVER w - avg(price) OVER w * avg(price) OVER w, 0)) AS stddev20
FROM trades
WINDOW w AS (PARTITION BY symbol ORDER BY timestamp ROWS 19 PRECEDING);

SELECT timestamp, symbol, sma20 - 2 * stddev20 AS lower_band, sma20 + 2 * stddev20 AS upper_band
FROM trades_bollinger WHERE symbol = 'BTC-USDT' LIMIT -10;
```

**Anchored windows** reset at a session boundary (session VWAP, intraday
stats). They must be named windows:

```sql
WINDOW s AS (
  PARTITION BY symbol ORDER BY timestamp
  ANCHOR DAILY '09:30' 'America/New_York'      -- or ANCHOR EXPRESSION expr
)
```

Function support:

| Function | Bounded ROWS | Bounded RANGE | Anchored |
|----------|--------------|---------------|----------|
| `avg`, `count`, `sum`, `min`, `max`, `first_value`, `nth_value`, `lag` | yes | yes | yes |
| `ksum` | yes | no | yes |
| `last_value` | yes | yes | yes (not with IGNORE NULLS) |
| `row_number`, `rank`, `dense_rank` | no | no | yes |
| `stddev*`, `var*`, `corr`, `covar*` | no | no | yes |
| EMA / VWEMA `avg(value, kind, param [, volume])` | no | no | yes |

Rules:
- One WAL base table: no JOINs, subqueries, CTEs, views or other live views.
  A materialized view can be the base.
- Project the designated timestamp plainly under its own name. No `SELECT *`.
- No `SAMPLE BY`, `GROUP BY`, `LATEST ON`, `DISTINCT`, `LIMIT`, no `WHERE` on
  the designated timestamp, no non-deterministic functions.
- `PARTITION BY` is required in every window. VARCHAR/UUID/DECIMAL/array keys
  are rejected; cast with `venue::symbol`. No `UNBOUNDED PRECEDING` unless
  anchored. `lead`, `percent_rank`, `cume_dist`, `ntile` are rejected.
- Rolling stddev over a bounded frame is rejected: use the
  `sqrt(avg(x*x) - avg(x)^2)` formula above.
- Freshness: reads see refreshed rows before the flush, except `LATEST ON`,
  bases with DEDUP, and replicas (up to one `FLUSH EVERY` behind).
  Out-of-order base rows rewrite rows already emitted.

```sql
DROP LIVE VIEW [IF EXISTS] v;
ALTER LIVE VIEW v RESUME WAL;
SHOW CREATE LIVE VIEW v;
SELECT view_name, base_table_name, view_status, lag_seqtxn, lag_micros FROM live_views();
```

No TTL, no INSERT/UPDATE. An invalid live view is permanent: save its
`SHOW CREATE LIVE VIEW` output and recreate it.

---

## Deduplication: UPSERT or SKIP

```sql
CREATE TABLE ... DEDUP UPSERT KEYS(timestamp, symbol);   -- latest write wins
CREATE TABLE ... DEDUP SKIP KEYS(timestamp, symbol);     -- first write wins (10.2.0)

ALTER TABLE trades DEDUP ENABLE UPSERT KEYS(timestamp, symbol);
ALTER TABLE trades DEDUP ENABLE SKIP KEYS(timestamp, symbol);   -- switch mode, no disable needed
ALTER TABLE trades DEDUP DISABLE;
```

| | UPSERT | SKIP |
|---|---|---|
| New row matches a stored row | replaces it | is discarded |
| Duplicates within one commit | last kept | first kept |
| PostgreSQL equivalent | `ON CONFLICT DO UPDATE` | `ON CONFLICT DO NOTHING` |

- WAL tables only. Keys must include the designated timestamp; arrays cannot
  be keys.
- Use SKIP for immutable event logs where a replay must not overwrite the
  original; UPSERT for corrections and snapshots.
- Versions before 10.2.0 treat a SKIP table as UPSERT and suspend on a logged
  `DEDUP ENABLE SKIP`: upgrade replicas first.

---

## Retention and Storage Tiers

### TTL (open source and Enterprise tables, and materialized views)

```sql
CREATE TABLE ... TIMESTAMP(ts) PARTITION BY DAY TTL 30 DAYS;
ALTER TABLE trades SET TTL 4 WEEKS;
ALTER TABLE trades SET TTL 0h;            -- remove
SELECT table_name, ttlValue, ttlUnit FROM tables();
```

TTL drops whole partitions and must be a whole multiple of the partition size.
On Enterprise, a table TTL becomes `STORAGE POLICY(DROP LOCAL ...)`; use
storage policies there.

### Storage Policy (Enterprise)

```sql
CREATE TABLE ... TIMESTAMP(ts) PARTITION BY DAY
  STORAGE POLICY(TO PARQUET 3d, DROP LOCAL 1M);

ALTER TABLE trades SET STORAGE POLICY(
  TO PARQUET 7 DAYS, TO REMOTE 14 DAYS, DROP LOCAL 30 DAYS, DROP REMOTE 7 YEARS);
ALTER TABLE trades ENABLE | DISABLE | DROP STORAGE POLICY;
SELECT * FROM storage_policies;
```

- `TO PARQUET`: convert locally, remove native files.
- `TO REMOTE`: upload a copy to cold storage; local stays writable.
- `DROP LOCAL`: seal read-only and remove the local copy; reads move to remote
  if `TO REMOTE` is set, otherwise the data is gone.
- `DROP REMOTE`: delete from the object store after a grace period.
- Order: `TO PARQUET <= DROP LOCAL`, `TO REMOTE <= DROP LOCAL <= DROP REMOTE`.
  `SET` replaces the whole policy. Mutually exclusive with TTL (clear it with
  `SET TTL 0 HOURS`). Not allowed on materialized views.

**Cold storage** (Enterprise): partitions in S3, GCS, Azure Blob or a
filesystem store, as Parquet, still queried with normal SQL. Needs
`cold.storage.enabled=true` on every instance and a WAL table. `DROP LOCAL`
makes a partition permanently read-only. See
`https://questdb.com/docs/concepts/cold-storage.md`.

### Parquet

```sql
-- Store new partitions as Parquet (partitioned WAL tables only)
CREATE TABLE sensors (ts TIMESTAMP, id SYMBOL, temp DOUBLE)
  TIMESTAMP(ts) PARTITION BY DAY FORMAT PARQUET;
ALTER TABLE sensors SET FORMAT NATIVE;           -- affects partitions written afterwards

-- Manual conversion (open source): older partitions, async
ALTER TABLE trades CONVERT PARTITION TO PARQUET WHERE timestamp < '2025-08-31';
ALTER TABLE trades CONVERT PARTITION TO PARQUET WHERE timestamp < '2025-08-31'
  WITH (bloom_filter_columns = 'symbol,side', bloom_filter_fpp = 0.01);
ALTER TABLE trades CONVERT PARTITION TO NATIVE WHERE timestamp < '2025-08-31';
SHOW PARTITIONS FROM trades;                     -- isParquet, parquetFileSize

-- Per-column encoding
temperature DOUBLE PARQUET(rle_dictionary, zstd(3))

-- Read external Parquet; TIMESTAMP() makes SAMPLE BY / ASOF work
SELECT timestamp, avg(price) FROM read_parquet('trades.parquet') TIMESTAMP(timestamp) SAMPLE BY 1h;
CREATE TABLE trades2 AS (SELECT * FROM read_parquet('trades.parquet'))
  TIMESTAMP(timestamp) PARTITION BY DAY;
```

- The active (latest) partition is never converted.
- `UPDATE` is not supported on Parquet partitions; DEDUP upserts work.
- Out-of-order writes into Parquet partitions are more expensive.
- Parquet partitions are the recommended way to compress data (instead of ZFS).
- Export: `GET /exp?query=...&fmt=parquet`.

---

## Indexes

There is no `CREATE INDEX`. SYMBOL columns are **not** indexed by default.
Add an index only for selective filters on a high-cardinality symbol:

```sql
symbol SYMBOL INDEX                                         -- bitmap, in CREATE TABLE
symbol SYMBOL INDEX TYPE POSTING INCLUDE (price, amount)    -- posting + covering
ALTER TABLE trades ALTER COLUMN symbol ADD INDEX TYPE POSTING INCLUDE (price);
ALTER TABLE trades ALTER COLUMN symbol DROP INDEX;
```

Posting indexes are about 13x smaller than bitmap and read faster; covering
columns let a filtered query skip the column files. Opt out per query with
`/*+ no_covering */` or `/*+ no_index */`.
