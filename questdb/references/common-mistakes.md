# QuestDB Common Mistakes

Patterns that LLMs get wrong because they exist in PostgreSQL but not in
QuestDB, because QuestDB's syntax differs, or because older QuestDB versions
behaved differently. Read this before writing novel QuestDB SQL.

## SQL Syntax

| Don't | Do instead | Why |
|---|---|---|
| `GROUP BY time_bucket('5m', ts)` | `SAMPLE BY 5m` | `time_bucket()` does not exist. |
| `date_trunc('hour', ts)` in a GROUP BY | `SAMPLE BY 1h`, or `timestamp_floor('1h', ts)` | `date_trunc` exists, but SAMPLE BY is the time-bucketing primitive. |
| `SELECT DISTINCT ON (symbol) ...` | `... LATEST ON ts PARTITION BY symbol` | `DISTINCT ON` does not exist. For other orderings use `row_number()` in a CTE. |
| `... HAVING avg(price) > 100` | Subquery or CTE, then `WHERE` | `HAVING` does not exist. |
| `... QUALIFY row_number() OVER (...) = 1` | CTE, then `WHERE rn = 1` | `QUALIFY` does not exist. |
| `INTERVAL '1 hour'`, `now() - INTERVAL '1 day'` | TICK `'$now-1d..$now'`, or `dateadd('d', -1, now())` | No PostgreSQL interval literals. |
| `ORDER BY ts NULLS LAST` | `ORDER BY ts` | `NULLS FIRST/LAST` is not supported. |
| `OFFSET 10 LIMIT 20` | `LIMIT 10, 30` | `LIMIT lo, hi`. `LIMIT -10` gives the last 10 rows. |
| `SELECT ... FROM generate_series(...)` for time slots | TICK lists or `SAMPLE BY ... FROM ... TO ... FILL(...)` | Use QuestDB's interval and fill features for time grids. `generate_series()` is fine in `FROM` for integer sequences. |
| Correlated subquery in `WHERE` / `EXISTS` | `LATERAL JOIN`, `ASOF JOIN`, or a window function | Correlated subqueries are not supported. |
| `col IN (SELECT ...)` where `col` is not a SYMBOL | `JOIN (SELECT DISTINCT col FROM ...) d ON t.col = d.col`; for NOT IN, `LEFT JOIN` the same and `WHERE d.col IS NULL` | `IN (subquery)` works only on a SYMBOL column. VARCHAR and UUID fail with `constant expected`; LONG, DOUBLE and TIMESTAMP with `cannot compare ... with type CURSOR`. |
| `SELECT book, count_distinct(px) ... GROUP BY book` with a DOUBLE `px` | `count_distinct(px::varchar)`, or `SELECT book, count() FROM (SELECT DISTINCT book, px FROM t) GROUP BY book` | Keyed `count_distinct` has no DOUBLE, FLOAT, DECIMAL or BOOLEAN variant. Without keys it works. |
| `... HORIZON JOIN q ... AS h JOIN dim d ON ...` (any other join before or after) | HORIZON in a CTE grouped by the key, then `JOIN dim` on the CTE | `only horizon joins can follow a horizon join` / `horizon join cannot be combined with other joins`. |
| `FROM (SELECT ... FROM t JOIN dim ...) x HORIZON JOIN ...` | Single-table left side (filters, projections, computed columns are fine); join the dimension after aggregating | `left-hand side of HORIZON JOIN can only be a table with an optional filter`. The right side has the same rule. |
| `FROM tables() t CROSS JOIN table_columns(t.table_name)` | `SELECT table_name, column_name, data_type FROM information_schema.columns` | `table_columns()` needs a constant name. `information_schema` reports PostgreSQL types (SYMBOL shows as `character varying`); use `table_columns('t')` per table when the QuestDB type matters. |
| `ntile(4) OVER (PARTITION BY c ORDER BY abs(qty))` | `SELECT abs(qty) AS aq, ntile(4) OVER (PARTITION BY c ORDER BY aq)` | Window `ORDER BY` takes columns or aliases, not expressions (`Invalid column: abs`). `PARTITION BY` accepts expressions. |
| `WITH f AS (SELECT ...) timestamp(ts)` | `WITH f AS ((SELECT ... ORDER BY ts) timestamp(ts))` | `timestamp()` goes inside the CTE body, on a parenthesised ordered subquery. |
| `CASE WHEN uuid_col IS NULL ...`, `CASE WHEN uuid_col = '...' ...` | `CASE WHEN uuid_col IS NOT NULL THEN 0 ELSE 1 END`, `sum((uuid_col IS NULL)::int)`, `count() - count(uuid_col)`, or `uuid_col::varchar = '...'` | `IS NULL` and `=` on UUID or LONG256 inside CASE fail with `type UUID is not supported in 'switch'`. `IS NOT NULL` and `!=` work. |
| `UNPIVOT (...)` | `UNION ALL` | No UNPIVOT keyword. PIVOT exists. |
| `DELETE FROM t WHERE ...` | `ALTER TABLE t DROP PARTITION ...`, TTL, or DEDUP corrections | No row-level DELETE. |
| `STRING` or `TEXT` | `VARCHAR` (unique) or `SYMBOL` (repeated) | `STRING` is legacy. |
| `ADD CONSTRAINT`, `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL` | DEDUP keys for uniqueness | No constraints. |
| `CREATE INDEX ON t(col)` | `ALTER TABLE t ALTER COLUMN col ADD INDEX [TYPE POSTING INCLUDE (...)]` | No CREATE INDEX. SYMBOL columns are not indexed by default; index only selective filters. |
| `INSERT ... ON CONFLICT DO UPDATE` | `DEDUP UPSERT KEYS(ts, ...)` on the table | Declared on the table, not per insert. |
| `INSERT ... ON CONFLICT DO NOTHING` | `DEDUP SKIP KEYS(ts, ...)` (10.2+) | First write wins. |
| `?` placeholders | `$1`, `$2` | The server does not accept `?`. |

## Time Filters (TICK)

| Don't | Do instead | Why |
|---|---|---|
| `WHERE ts >= dateadd('h', -1, now())` | `WHERE ts IN '$now-1h..$now'` | TICK is preferred: shorter, compiles to interval scans. |
| `WHERE ts BETWEEN '2025-01-01' AND '2025-01-31'` | `WHERE ts IN '[2025-01]'` | Month and year dates expand to every day. |
| `'2025-01-[01..31]'` | `'[2025-01]'` | Imprecise dates replace day ranges. |
| `WHERE ts = '$today'` | `WHERE ts IN '$today'` | TICK needs `IN`. |
| `WHERE ts IN $now - 1h` | `WHERE ts IN '$now-1h..$now'` | Quote it; a range needs `..`. |
| `'$now - 1 hour'`, `'$now - 5 business days'` | `'$now-1h'`, `'$now-5bd..$now'` | Single-letter units; `bd` = business days. |
| `WHERE ts IN '$now'` | `'$now-5m..$now'` or `'$now;5m'` | `$now` alone matches one microsecond. |
| `'$now-@lookback..$now'` | `@w := '$now-1h..$now'` then `IN @w`, or `IN interval(dateadd(...), @asof)` | Variables are not substituted inside a string. Store the whole TICK string, or use `interval()` for computed bounds. |
| `"query=... '$now-1h..$now'"` in a double-quoted shell string | Escape: `'\$now-1h..\$now'` | The shell expands `$now` to an empty string. |
| `'2025-01-10..2025-01-12'`, `'2025-01-10T00:00..2025-01-12T00:00'` | `'2025-01-[10..11]'`, `'2025-01-10;2d'`, `'[2025-01-10, 2025-01-11]'`, `interval('2025-01-10', '2025-01-12')` | `..` only works between `$` variables. Mixing a variable and a date fails too. |
| `'$today-2d..$today'` for "the last two days" | `'$today-2d..$yesterday'` | A day end bound includes the whole day, so the first form is three days. |
| `100ms` (TICK, SAMPLE BY, HORIZON, TOLERANCE) | `100T` | No `ms` unit; `T` is milliseconds. |
| `'$now-100U..$now'` | `'$now-100u..$now'` | TICK microseconds are lowercase `u`. `U` raises no error and breaks the bound it is on: here the start becomes the beginning of the table. SAMPLE BY, HORIZON and TOLERANCE are the opposite: `U`, and they reject `u`. |

## SAMPLE BY

| Don't | Do instead | Why |
|---|---|---|
| `SAMPLE BY 5m ALIGN TO CALENDAR` | `SAMPLE BY 5m` | Calendar alignment is the default since 7.4. Write ALIGN only with `TIME ZONE` or `WITH OFFSET`. |
| `SAMPLE BY '5 minutes'`, `SAMPLE BY 5 m` | `SAMPLE BY 5m` (or `SAMPLE BY m`) | Unit suffix, no quotes, no space. |
| `SAMPLE BY 5m ALIGN TO CALENDAR FILL(PREV)` | `SAMPLE BY 5m FILL(PREV) ALIGN TO CALENDAR TIME ZONE 'Europe/London'` | FILL comes before ALIGN. |
| `GROUP BY symbol` next to SAMPLE BY | Just select `symbol` | Non-aggregated columns are keys automatically. |
| Window function in the same SELECT as SAMPLE BY | SAMPLE BY in a CTE, window outside | Cannot mix at one level. |
| `FILL(LINEAR)` on a keyed query | `FILL(PREV)` / `FILL(NULL)` | LINEAR is not supported with keys. |
| Expecting local-time output from `TIME ZONE` | `to_timezone(ts, 'tz')` on the result | Output timestamps are UTC. |

## Materialized and Live Views

| Don't | Do instead | Why |
|---|---|---|
| `CREATE MATERIALIZED VIEW v AS (...) REFRESH EVERY 5m` | `CREATE MATERIALIZED VIEW v REFRESH EVERY 5m AS (...)` | REFRESH goes before AS. |
| `ALTER MATERIALIZED VIEW v INVALIDATE` | `REFRESH MATERIALIZED VIEW v FULL` | There is no INVALIDATE command. |
| Assuming no WHERE or JOIN is allowed | WHERE and JOIN (with `WITH BASE`) work in aggregating views | Old restriction. |
| `LIMIT` or `row_number() OVER (...) + 1` on the base table in a view (10.2+) | Remove it, or move it to the query that reads the view | Rejected since 10.2.0. |
| A mat view only to copy filtered rows, with a fake `SAMPLE BY` | Passthrough view: `AS (SELECT ... FROM t WHERE ...)` without SAMPLE BY | 10.2.0 supports non-aggregating views. |
| `LATEST ON` view to keep last value per key | `EXPIRE ROWS KEEP LATEST ON ts PARTITION BY key` | LATEST ON is rejected in passthrough views. |
| `INSERT INTO mat_view` | Insert into the base table | Views refresh from their base. |
| `DROP TABLE mat_view` | `DROP MATERIALIZED VIEW mat_view` (`DROP LIVE VIEW`, `DROP VIEW`) | Each kind has its own DROP. |
| `WITH BASE` on a single-table view | Remove it | Only needed with JOINs. |
| `STORAGE POLICY` on a materialized view | `TTL` on the view | Policies apply to tables only. |
| `stddev()` over a bounded frame in a live view | `sqrt(greatest(avg(x*x) OVER w - avg(x) OVER w * avg(x) OVER w, 0))` | Live views support stddev only in anchored windows. |
| `SELECT *` or a timestamp `WHERE` in a live view | Project columns explicitly; filter when reading the view | Rejected in live view definitions. |

## Window Functions

| Don't | Do instead | Why |
|---|---|---|
| `WHERE rn = 1` next to `row_number() OVER (...) AS rn` | CTE, filter outside | Cannot filter window results at the same level. |
| Manual EMA recurrence | `avg(price, 'period', 20) OVER (ORDER BY ts)` | Native EMA. Also `'alpha'` and time constants. |
| `avg(x, 'period', 20) OVER (ORDER BY ts ROWS ...)` | Drop the frame | EMA and VWEMA reject ROWS/RANGE clauses. |
| `sqrt(avg(x*x) - avg(x)^2)` workaround for Bollinger in queries | `stddev_samp(x) OVER (... ROWS BETWEEN 19 PRECEDING AND CURRENT ROW)` | Windowed stddev works in queries (not in live views). |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | `OVER (ORDER BY ts CUMULATIVE)` | Shorthand; both work. |
| `SELECT ts AS time, avg(x) OVER (ORDER BY ts)` | Window in a CTE, alias `ts AS time` outside | After aliasing, `ts` is not visible to OVER. |

## Arrays

| Don't | Do instead | Why |
|---|---|---|
| `bids[0]`, `bids[0][0]` | `bids[1]`, `bids[1][1]` | 1-indexed. `bids[1]` prices, `bids[2]` sizes, `bids[1][1]` best bid. |
| Four 1D arrays for a book | `bids DOUBLE[][]`, `asks DOUBLE[][]` | One 2D array per side. |
| Python lists for array columns | `np.array(..., dtype=np.float64)`, shape `(2, N)` | Only float64 numpy arrays are accepted. |

## Ingestion and Clients

| Don't | Do instead | Why |
|---|---|---|
| `from questdb.ingress import Sender` with `tcp::addr=...:9009` for new code | `questdb.connect("ws::addr=localhost:9000;")` and `db.sender()` | QWP is the native protocol since 10.0; ILP is a compatibility protocol. |
| `questdb.connect("http::...")` | `ws::` / `wss::` | `connect()` only accepts QWP strings. |
| `protocol_version=2` in a `ws::` string | Omit it | QWP negotiates its version. |
| `wss::` assuming port 443 | `wss::addr=host:9000;` (or the real port) | Both schemes default to 9000. |
| A new connection per row or per callback | One handle per process, long-lived sender lease | Connections are pooled and expensive to open. |
| `sender.flush()` after every row | Rely on auto-flush (1,000 rows / 100 ms) or flush per batch | Per-row flushes kill throughput. |
| Assuming `flush()` means stored | `flush(wait=True)`; then `wait_wal_table('t')` before reading | Flush returns before the ack; reads see rows after WAL apply. |
| `TimestampNanos.now()` for events you may replay | The event's own timestamp, plus DEDUP keys | Server-side times change on replay and create duplicates. |
| `INSERT INTO ... VALUES` in a loop for streams | QWP sender | INSERT is for small volumes. |
| HTTP POST to `/exec` | `GET /api/v1/sql/execute?query=...` | GET only; `/exec` is the deprecated name. |
| Hand-encoding `()` in curl URLs | `curl -G --data-urlencode "query=..."` | Encodes everything correctly. |
| `import psycopg2`, `dbname=questdb` | `import psycopg`, `dbname=qdb` | psycopg v3; database name is ignored, `qdb` by convention. |
| Multi-host failover on open source | Enterprise only; OSS is single-node | Failover needs replication. |

## Schema Design

| Don't | Do instead | Why |
|---|---|---|
| `VARCHAR` for tickers, venues, sides | `SYMBOL` | Dictionary-encoded, fast to filter and group. |
| Table without `TIMESTAMP(col)` | Always designate a timestamp | Needed for SAMPLE BY, LATEST ON, joins, TTL, DEDUP. |
| `... PARTITION BY DAY WAL` | `... PARTITION BY DAY` | WAL is the default for partitioned tables. |
| No `PARTITION BY` | Pick by volume (30-80M rows per partition) | Unpartitioned tables are not WAL and cannot use TTL. |
| `SYMBOL CAPACITY 256` | `SYMBOL` | Capacity is automatic since 9.0. |
| Separate date and time columns | One `TIMESTAMP` | Time-series features need one designated timestamp. |
| No DEDUP on tables behind failover or replay | `DEDUP UPSERT KEYS(ts, ...)` or `SKIP KEYS` | At-least-once delivery creates duplicates otherwise. |
| `TTL` on Enterprise tables | `STORAGE POLICY(TO PARQUET 7d, DROP LOCAL 1M)` | Enterprise translates TTL to a policy and rejects `ALTER TABLE SET TTL`. |
| ZFS for compression | Parquet partitions (`FORMAT PARQUET`, `CONVERT PARTITION`, or a storage policy) | The recommended way to compress data. |

## Web Console Dashboards (MCP)

| Don't | Do instead | Why |
|---|---|---|
| Grafana by default | Web Console notebook via the MCP server | No extra service; Grafana only on request. |
| Calling tools before pairing, or pairing without showing `userMessage` | `get_pairing_credentials`, show `userMessage` verbatim, then `wait_for_pairing` | The user needs the link or token. |
| Many `add_cell` + `set_cell_*` calls for a new dashboard | One `apply_notebook_state` | Atomic and much faster. |
| Omitting existing cells or `chart_config` in `apply_notebook_state` | Send every cell; `preserve_value: true` for untouched SQL; re-send chart configs | It is a full PUT: omissions delete. |
| `chart_config.queries` shorter than the statement count | One entry per `;` statement | Rejected otherwise. |
| Candlestick without `ohlc` | `ohlc: {open, high, low, close}` | Required. |
| No refresh setting on a live dashboard | `auto_refresh_default` and per-cell `auto_refresh` | Nothing polls until set. |
| DDL in dashboard cells | Create tables and views separately | DDL cells never auto-run or refresh. |
| Volume bars on the price axis | Second query on `axis: "right"` with `right_axis` max pinned (e.g. 0..4 for 0..1 values), or the candlestick `volume` column | Keeps candles readable. |
| `activate_notebook` right after creating | Tell the user, switch only if they agree | The user decides when to change tabs. |
