---
name: questdb
description: >
  Use this skill whenever working with QuestDB, the high-performance
  time-series database. Trigger on any mention of QuestDB, time-series SQL
  with SAMPLE BY, LATEST ON, ASOF/HORIZON/WINDOW/LATERAL JOIN, TICK interval
  syntax, materialized or live views, DEDUP, TTL or storage policies,
  ingestion or querying with the QuestDB clients (QWP over WebSocket, ILP,
  PGWire, REST) in Python, Java, Go, Rust, C/C++, .NET, Node.js, Ruby or the
  browser, building dashboards in the QuestDB Web Console through its MCP
  server, Grafana on QuestDB, order book or market data analytics, or
  monitoring a QuestDB instance with sys.metrics. QuestDB extends SQL with
  time-series keywords and standard PostgreSQL patterns often fail, so read
  this skill before writing QuestDB SQL or client code.
---

# QuestDB Skill

Last updated for QuestDB 10.2 (open source) and QuestDB Enterprise 4.2. For
features added in later releases, check the
[release notes](https://questdb.com/release-notes/) and the
[docs changelog](https://questdb.com/docs/changelog.md).

## How to Use This Skill

This file has the rules and syntax you need for most tasks. Read a reference
file only when the task needs it:

| Task | Read |
|------|------|
| Ingest or query from code, connect strings, REST, PGWire, ILP | `references/clients.md` |
| Bulk export as fast as possible, parallel backfill, replay, generators, query-shape performance tips | `references/patterns.md` |
| Build a dashboard or notebook in the Web Console (preferred over Grafana) | `references/dashboards-mcp.md` and its `references/examples/*.json` |
| Materialized views (passthrough, EXPIRE ROWS), live views, views, DEDUP, TTL, storage policy, Parquet, indexes | `references/views-and-storage.md` |
| Monitor an instance (`sys.metrics`, WAL telemetry, meta functions) | `references/monitoring.md` |
| Any SQL beyond the examples in this file: read it before writing the query | `references/common-mistakes.md` |
| "Does QuestDB have function X?", signatures, keywords, types | `references/sql-grammar.md` |
| Indicators (MACD, ATR, Stochastic, Keltner, OBV, drawdown, ...) | `references/indicators.md` |
| Finance/TCA or time-series recipes (markout, slippage, IS, OFI, FILL patterns, subtotals, ...) | `references/cookbook.md` |
| Open-ended analysis: find issues, anomalies or patterns in a dataset | `references/cookbook.md` (Analysing a Dataset) |
| End-to-end crypto feed pipeline (cryptofeed + QuestDB) | `references/crypto-pipeline.md` |
| Grafana, only when the user asks for it | `references/grafana.md` |
| Enterprise: auth, tokens, RBAC, replication, failover, cold storage | `references/enterprise.md` |

Working style:
- Write the files and run them. Do not explore library sources, check
  package versions or re-verify infrastructure the user said is running.
- Do not read reference files for topics already covered here, except
  `references/common-mistakes.md`, which applies to every query you write.
- When you need syntax not covered here or in a reference, fetch the doc page
  as markdown (below) instead of guessing.

## Defaults That Matter

1. **QuestDB is not PostgreSQL.** No `HAVING`, `QUALIFY`, `DISTINCT ON`,
   `NULLS FIRST/LAST`, `INTERVAL '1 hour'`, `time_bucket()`, `CREATE INDEX`,
   constraints, triggers or row `DELETE`.
2. **Time filters use TICK**: `WHERE ts IN '$now-1h..$now'`, `IN '$today'`,
   `IN '[2025-01]'`. Not `ts >= dateadd(...)`, not `BETWEEN`, not
   `now() - INTERVAL`. Keep `dateadd()` / `interval()` only for bounds
   computed from variables or columns.
3. **`SAMPLE BY` aligns to calendar by default.** Never write
   `ALIGN TO CALENDAR` unless you also add `TIME ZONE` or `WITH OFFSET`.
4. **Ingest and query with QWP clients** (`ws::addr=host:9000;`), one handle
   for both. ILP is a compatibility protocol for existing pipelines; REST is
   for small ad-hoc reads; PGWire for BI tools.
5. **Dashboards go in the Web Console** via the QuestDB MCP server (notebooks
   with live charts, variables, per-cell refresh). Grafana only on request.
6. **Every time-series table has a designated timestamp, partitioning, and
   DEDUP keys** when replays are possible. WAL is the default; do not write
   the `WAL` keyword.
7. **Retention:** `TTL` on open source (and on materialized views
   everywhere); `STORAGE POLICY` on Enterprise tables.
8. Use **SYMBOL** for repeated strings, **VARCHAR** for unique strings
   (`STRING` is legacy). Arrays are **1-indexed**.
9. **Valid-looking SQL that fails.** These come up most; the full list, with
   workarounds, is in `references/common-mistakes.md`:
   - `col IN (SELECT ...)` works only when `col` is a SYMBOL. For UUID,
     VARCHAR or numbers, `JOIN (SELECT DISTINCT ...)` instead.
   - A HORIZON JOIN cannot be combined with other joins, and each side must
     be a single table. Join dimension tables after aggregating.
   - Keyed `count_distinct()` rejects DOUBLE: `count_distinct(px::varchar)`.
   - `CASE WHEN uuid_col IS NULL` fails: `CASE WHEN uuid_col IS NOT NULL
     THEN 0 ELSE 1 END`.
   - TICK `..` works only between `$` variables, not literal dates. There is
     no `ms` unit: write `100T`.

## Live Documentation

Every docs page has a plain-markdown twin: append `.md`.

```bash
curl -s "https://questdb.com/docs/query/sql/horizon-join.md"
curl -s "https://questdb.com/docs/llms.txt"      # index of every page with its .md URL
```

The demo instance `https://demo.questdb.io` (latest release, read-only) runs
any query over REST, which is a quick way to check syntax:

```bash
curl -sG --data-urlencode "query=SELECT * FROM fx_trades LIMIT 3" https://demo.questdb.io/api/v1/sql/execute
```

To check that a function exists on the server you are using, and its exact
signature: `SELECT name, signature_translated, type FROM functions() WHERE name ILIKE '%vwap%'`.

In a double-quoted shell string, escape TICK variables (`'\$now-1h..\$now'`)
or the shell expands `$now` to nothing.

---

## SQL Reference

### CREATE TABLE

```sql
CREATE TABLE IF NOT EXISTS trades (
    ts TIMESTAMP,
    symbol SYMBOL,
    side SYMBOL,
    price DOUBLE,
    amount DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY TTL 30 DAYS
DEDUP UPSERT KEYS(ts, symbol);
```

Full clause order:
`TIMESTAMP(col) PARTITION BY unit [TTL n unit | STORAGE POLICY(...)] [FORMAT NATIVE|PARQUET] [BYPASS WAL] DEDUP {UPSERT|SKIP} KEYS(...)`.

- `TIMESTAMP(col)` is required for SAMPLE BY, LATEST ON, time-series joins,
  TTL and DEDUP. It and the partitioning cannot change after creation.
- Partition target: 30-80 million rows per partition. Rough guide:
  more than 1B rows/day `HOUR`, 30-500M `DAY`, 5-30M `WEEK`, 1-5M `MONTH`,
  less `YEAR`.
- `DEDUP UPSERT KEYS` (latest write wins) or `DEDUP SKIP KEYS` (first write
  wins, 10.2+). Keys must include the designated timestamp.
- `FORMAT PARQUET` stores new partitions as Parquet (partitioned WAL tables).
- `SYMBOL CAPACITY` is obsolete; symbol capacity is automatic.
- Types: BOOLEAN, BYTE, SHORT, INT, LONG, FLOAT, DOUBLE, DECIMAL(p,s), CHAR,
  VARCHAR, SYMBOL, TIMESTAMP (µs), TIMESTAMP_NS, DATE, UUID, IPv4, LONG256,
  GEOHASH, BINARY, arrays `DOUBLE[]`, `DOUBLE[][]`.
- `ALTER TABLE t ADD COLUMN c SYMBOL`, `DROP COLUMN`, `RENAME COLUMN a TO b`,
  `ALTER COLUMN c TYPE VARCHAR`, `SET TTL`, `DEDUP ENABLE ... / DISABLE`,
  `CONVERT PARTITION TO PARQUET WHERE ts < '...'`.
- `INSERT INTO ... VALUES` is for low volume and tests only.

### Timestamp Filtering: TICK

TICK is QuestDB's interval syntax. It compiles to interval scans, handles
time zones, business days and exchange calendars, and replaces
`UNION ALL` of several ranges.

Order: `date [T time] @timezone #dayFilter ;duration`

```sql
WHERE ts IN '$today'                          -- full day; also $yesterday, $tomorrow
WHERE ts IN '$now-1h..$now'                   -- last hour (spaces optional: '$now - 1h..$now')
WHERE ts IN '$now-5bd..$now'                  -- last 5 business days
WHERE ts IN '$now;1h'                         -- one hour forward from now
WHERE ts IN '2025-01-15T09:30;6h30m'          -- a session
WHERE ts IN '[2025-01]'                       -- all of January
WHERE ts IN '[2025]T09:30;1h'                 -- 09:30-10:30 every day of 2025
WHERE ts IN '2025-01-[10..15]'                -- days 10 to 15
WHERE ts IN '2025-[01..03]T09:30'             -- every day Jan-Mar at 09:30
WHERE ts IN '[2025-01]#workday'               -- weekdays (#weekend, #Mon,Wed,Fri)
WHERE ts IN '[2025-01]T09:30@America/New_York#workday;6h30m'
WHERE ts IN '2025-01-15T[09:00,14:30];1h'     -- two windows in one day
WHERE ts IN '[$today, $yesterday, 2025-01-15]'
WHERE ts IN '2025-W01-[1..5]T09:00;8h'        -- ISO week
WHERE ts IN '[2025-01]#XNYS'                  -- Enterprise: NYSE sessions, holidays, early closes
```

Rules:
- Use `IN`, never `=`. Always single-quoted.
- `$now` alone is one microsecond: add a range (`..`) or a duration (`;`).
- `..` needs a `$` variable on both sides. Between literal dates it fails
  (`'2025-01-10..2025-01-12'` is an invalid date); use `'2025-01-[10..11]'`,
  `'2025-01-10;2d'`, `'[2025-01-10, 2025-01-11]'` or
  `interval('2025-01-10', '2025-01-12')`.
- A day variable as the end bound includes that whole day:
  `'$today-2d..$today'` is three days. For two complete days use
  `'$today-2d..$yesterday'`.
- Units: `y M w d bd h m s T`(ms) `u`(µs) `n`(ns). Case-sensitive:
  `M` months, `m` minutes. `bd` only in arithmetic, not durations. No `ms`:
  write `100T`.
- Microseconds are lowercase `u` in TICK but uppercase `U` everywhere else
  (SAMPLE BY, HORIZON, TOLERANCE, which reject `u`). In TICK, `U` raises no
  error and gives wrong results: `'$now-100U..$now'` becomes an unbounded
  scan from the start of the table.
- Brackets are required for lists and for ranges with suffixes
  (`'[$now-2h..$now]@Europe/London'`).
- A TICK string can live in a variable:
  `DECLARE @w := '$now-1h..$now' SELECT ... WHERE ts IN @w`.
- For bounds computed from variables or columns, use `interval()`:
  `WHERE ts IN interval(dateadd('m', -1, @asof), @asof)`, or a plain
  `ts >= ... AND ts <= ...`.

### SAMPLE BY

```sql
SELECT ts, symbol,
       first(price) AS open, max(price) AS high, min(price) AS low,
       last(price) AS close, sum(amount) AS volume
FROM trades
WHERE ts IN '$today'
SAMPLE BY 15m;
```

```
SAMPLE BY [n]unit
  [ROLLUP(...) | CUBE(...) | GROUPING SETS (...)]
  [FROM ts TO ts]
  [FILL(NONE | NULL | PREV | PREV(col) | LINEAR | constant [, ...])]
  [ALIGN TO CALENDAR [TIME ZONE tz] [WITH OFFSET 'hh:mm'] | ALIGN TO FIRST OBSERVATION]
```

- Units `n U T s m h d w M y`; the count defaults to 1 (`SAMPLE BY h`). No
  quotes, no space: `SAMPLE BY 5m`.
- Non-aggregated columns (like `symbol`) become keys automatically. No
  `GROUP BY` needed.
- `FILL` comes before `ALIGN`. One fill per aggregate: `FILL(PREV, 0, NULL)`.
  `LINEAR` not for keyed queries. `FROM ... TO ...` emits empty buckets at the
  edges.
- Output timestamps are UTC even with `TIME ZONE`; convert with `to_timezone()`.
- `count_distinct(col)` instead of `COUNT(DISTINCT col)`. With keys (SAMPLE
  BY, GROUP BY, keyed joins) it rejects DOUBLE, FLOAT, DECIMAL and BOOLEAN:
  use `count_distinct(px::varchar)` or count over a `SELECT DISTINCT`.
- No `HAVING`: wrap in a subquery and filter with `WHERE`.

### LATEST ON

```sql
SELECT * FROM trades
WHERE ts IN '$now-1h..$now'
LATEST ON ts PARTITION BY symbol;
```

Most recent row per key. Replaces `DISTINCT ON` and `row_number() = 1`.
Reference the designated timestamp; add a time filter so it can stop early.

### DECLARE (Variables)

```sql
DECLARE
  @symbols := ('EURUSD', 'GBPUSD'),           -- value list for IN
  @window  := '$now-1h..$now',                -- TICK string
  @bar     := 1m
SELECT timestamp, symbol, avg(price)
FROM fx_trades
WHERE symbol IN @symbols AND timestamp IN @window
SAMPLE BY @bar;
```

- Substituted at parse time. Later variables can use earlier ones.
- One-item list needs a trailing comma: `('EURUSD',)`. Lists only on the
  right of `IN`. Binds: `@sym := $1` (never `?`).
- In views, `OVERRIDABLE @x := ...` makes a parameter callers can set.
- Web Console notebook variables are sent as this same `DECLARE` block.

### Joins

Pick the specialised join; defaulting to ASOF when HORIZON or WINDOW fits is
a common mistake.

| Question | Join |
|----------|------|
| Value at each row's time ("quote at trade time") | `ASOF JOIN` (`TOLERANCE 50T` to bound staleness) |
| Strictly before | `LT JOIN` |
| Interleave two series | `SPLICE JOIN` |
| Values at many offsets around each row (markout, IS, adverse selection) | `HORIZON JOIN` |
| Aggregate right rows in a ±window around each row | `WINDOW JOIN` |
| Per-row subquery: top-N per group, per-row LATEST ON / SAMPLE BY | `LATERAL JOIN` |

```sql
SELECT * FROM trades ASOF JOIN quotes ON (symbol) TOLERANCE 1s;

-- HORIZON: average mid at -1m..+5m every 30s after each trade, one parallel pass
SELECT h.offset / 1_000_000_000 AS horizon_sec, t.symbol,
       avg((m.best_bid + m.best_ask) / 2) AS avg_mid
FROM fx_trades t
HORIZON JOIN market_data m ON (symbol)
RANGE FROM -1m TO 5m STEP 30s AS h        -- or LIST (0, 1s, 30s, 5m) AS h
WHERE t.timestamp IN '$yesterday'
ORDER BY t.symbol, horizon_sec;

-- WINDOW: quotes within ±5s of each trade
SELECT t.symbol, t.timestamp, avg(c.bid_price) AS avg_bid, count() AS quotes
FROM fx_trades t
WINDOW JOIN core_price c ON (t.symbol = c.symbol)
  RANGE BETWEEN 5 seconds PRECEDING AND 5 seconds FOLLOWING
  EXCLUDE PREVAILING
WHERE t.symbol = 'EURUSD' AND t.timestamp IN '$now-1h..$now';

-- LATERAL: two largest fills per order
SELECT o.id, t.qty
FROM orders o
JOIN LATERAL (SELECT qty FROM fills WHERE order_id = o.id ORDER BY qty DESC LIMIT 2) t
ORDER BY o.id, t.qty DESC;
```

- Time-series joins need a designated timestamp on both sides.
  `ON (symbol)` when names match, `ON (a.sym = b.sym)` otherwise.
- **HORIZON** (10.1+): `h.offset` is in the left table's timestamp unit
  (ns for `TIMESTAMP_NS`); offset units `n U T s m h d` (uppercase `U`, no
  `ms`: `LIST (0, 100T)`). Without aggregates it
  returns one row per left row and offset. Right-side filters go in `ON`
  (`ON (t.symbol = c.symbol AND c.ecn = 'LMAX')`), which also works in Web
  Console notebooks; `(core_price WHERE ...) AS c` works on the server but
  stops notebook variables from applying. Several HORIZON JOINs can chain; only the
  last carries `RANGE`/`LIST ... AS`. No SAMPLE BY, window functions or other
  joins at the same level: wrap it.
- Each side of a HORIZON JOIN must be one table, optionally in a subquery that
  filters, projects or computes columns. A JOIN inside either side fails
  (`left-hand side of HORIZON JOIN can only be a table with an optional
  filter`). To add columns from another table, group the HORIZON by the key
  in a CTE and join the dimension table to that result.
- A left side without a designated timestamp (a table without one, or a CTE
  that lost it) needs `timestamp()` on an ordered subquery, inside the CTE
  body: `WITH f AS ((SELECT * FROM ledger ORDER BY ts) timestamp(ts))`.
  `WITH f AS (SELECT ...) timestamp(ts)` is a syntax error. This does not
  lift the single-table rule.
- **WINDOW**: units spelled out (`seconds`, `minutes`); bounds can be left
  columns. No GROUP BY or window functions at the same level.
- **LATERAL**: INNER, LEFT or CROSS only; the subquery can use outer columns,
  SAMPLE BY, LATEST ON, window functions, UNION ALL.
- Execution-quality queries (slippage, markout, IS, ECN scorecard) are all
  HORIZON JOIN recipes in `references/cookbook.md`.

### Window Functions

```sql
avg(close) OVER (PARTITION BY symbol ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma20
stddev_samp(close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sd20
sum(volume) OVER (ORDER BY ts CUMULATIVE) AS running_total
avg(price, 'period', 14) OVER (PARTITION BY symbol ORDER BY ts) AS ema14     -- EMA, no frame clause
avg(price, 'period', 14, volume) OVER (ORDER BY ts) AS vwema14
lag(close) OVER (PARTITION BY symbol ORDER BY ts) AS prev_close
avg(price) OVER (ORDER BY ts RANGE BETWEEN '1' HOUR PRECEDING AND CURRENT ROW)
```

- Also: `row_number`, `rank`, `dense_rank`, `percent_rank`, `ntile`,
  `cume_dist`, `lead`, `first_value`, `last_value`, `nth_value`, `ksum`,
  `var*`, `corr`, `covar*`, and `percentile_cont/disc(x, p) OVER (PARTITION BY ...)`.
- EMA kinds: `'period'`, `'alpha'`, or a time constant `'minute'`, `'hour'`, ...
  EMA rejects ROWS/RANGE frames.
- Windows cannot share a level with SAMPLE BY / GROUP BY, and you cannot
  filter on a window result at the same level: use a CTE.
- `WINDOW w AS (...)` names a window for reuse (`OVER w`).

### Views

| Need | Use |
|------|-----|
| Reusable query, parameters | `CREATE VIEW v AS (DECLARE OVERRIDABLE @x := ... SELECT ...)` |
| Incremental rollups | `CREATE MATERIALIZED VIEW v AS (SELECT ... SAMPLE BY 1m) PARTITION BY DAY TTL 30 DAYS` |
| Live subset of one table | Passthrough materialized view: same, without SAMPLE BY (10.2+) |
| Keep latest per key / top N / last 7 days | Materialized view + `EXPIRE ROWS KEEP LATEST ON ts PARTITION BY key` (10.2+) |
| Rolling window functions maintained per row | `CREATE LIVE VIEW v FLUSH EVERY 1s START FROM NOW AS SELECT ... OVER w ...` (beta) |

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS trades_ohlc_1m AS (
  SELECT ts, symbol, first(price) AS open, max(price) AS high,
         min(price) AS low, last(price) AS close, sum(amount) AS volume
  FROM trades SAMPLE BY 1m
) PARTITION BY DAY TTL 90 DAYS;
```

- `REFRESH IMMEDIATE` (default), `EVERY 5m`, `MANUAL`, `PERIOD (...)`: the
  clause goes before `AS`. JOINs need `WITH BASE base_table`.
- Aggregating views may use WHERE and JOIN. Since 10.2 the base-table part
  cannot use `LIMIT` or window functions.
- Cascade fine to coarse: `trades -> 1s -> 1m -> 1h`.
- Never INSERT into a view. `DROP MATERIALIZED VIEW`, not `DROP TABLE`.
- Full syntax and live view rules: `references/views-and-storage.md`.

### Reshaping and Reporting

```sql
-- Subtotals and grand total in one scan
SELECT symbol, side, sum(quantity) AS qty, GROUPING(symbol) AS g_sym
FROM fx_trades WHERE timestamp IN '$today'
GROUP BY ROLLUP(symbol, side);                   -- also CUBE(...), GROUPING SETS ((a,b),(a),())
-- with time buckets: ... SAMPLE BY 1h ROLLUP(symbol)

-- PIVOT
SELECT * FROM (SELECT symbol, side, quantity FROM fx_trades WHERE timestamp IN '$today')
PIVOT (sum(quantity) FOR side IN ('buy', 'sell') GROUP BY symbol);

-- Downsample for charts without inventing values
SELECT timestamp, price FROM fx_trades
WHERE symbol = 'EURUSD' AND timestamp IN '$today'
SUBSAMPLE lttb(price, 500);              -- m4/minmax(col, n), uniform(n), cadence(n), sdt(col, dev)

-- Arrays to rows; JSON arrays to typed rows
SELECT m.symbol, u.price, u.size, u.lvl
FROM market_data m, UNNEST(m.bids[1], m.bids[2]) WITH ORDINALITY u(price, size, lvl)
WHERE m.timestamp IN '$now-1s..$now';
SELECT u.* FROM UNNEST(payload::VARCHAR COLUMNS(id LONG, price DOUBLE, ts TIMESTAMP)) u;
```

- No UNPIVOT keyword: use `UNION ALL`.
- `LIMIT -10` returns the last 10 rows; `LIMIT 10, 30` is offset 10.
- `SELECT DISTINCT` works; `GROUP BY` is optional (implicit on
  non-aggregated columns).

### Key Functions

- Time: `now()`, `dateadd('m', -5, ts)`, `datediff('s', a, b)`,
  `timestamp_floor('15m', ts)`, `to_timezone(ts, 'Europe/London')`,
  `to_utc(ts, tz)`, `to_timestamp(str, fmt)`, `interval(lo, hi)`,
  `is_end_of_month(ts)`.
- Aggregates: `first`, `last`, `first_not_null`, `last_not_null`,
  `count_distinct`, `arg_max(v, k)`, `ksum`, `stddev_samp`, `corr`,
  `percentile_cont(x, 0.95)` / `percentile_disc(x, ARRAY[0.5, 0.99])` (exact),
  `approx_percentile(x, 0.99)`, `array_agg`, `kurtosis`, `skewness`,
  `regr_slope/intercept/r2`, `string_agg`.
- Finance: `twap(price, ts)`, `vwap(price, qty)`, `mid(bid, ask)`,
  `spread(bid, ask)`, `spread_bps(bid, ask)`, `wmid(bsz, bpx, apx, asz)`,
  `l2price(size, sizes[], prices[])`.
- Arrays (1-indexed; `bids[1]` prices, `bids[2]` sizes, `bids[1][1]` best bid):
  `array_cum_sum`, `array_sum`, `array_avg`, `insertion_point(arr, v)`,
  `array_position`, `dim_length`, element-wise `array_elem_sum/avg/min/max`.
- `SELECT wait_wal_table('t')` blocks until earlier writes are visible.
- Full catalogue: `references/sql-grammar.md`.

**Clause order:** `DECLARE → WITH → SELECT → FROM → JOIN → WHERE → LATEST ON →
SAMPLE BY / GROUP BY → WINDOW → SUBSAMPLE → ORDER BY → LIMIT`.

---

## Ingesting and Querying

Python (`pip install -U "questdb[dataframe]"`, Python 3.10+; the extra
brings pandas and pyarrow, which `to_pandas()` needs). Other languages, failover,
auth and exactly-once rules: `references/clients.md`.

```python
import questdb
from questdb import TimestampNanos

with questdb.connect("ws::addr=localhost:9000;") as db:
    db.execute("CREATE TABLE IF NOT EXISTS trades (ts TIMESTAMP, symbol SYMBOL, price DOUBLE) "
               "TIMESTAMP(ts) PARTITION BY DAY DEDUP UPSERT KEYS(ts, symbol)")
    with db.sender() as sender:
        sender.row("trades", symbols={"symbol": "BTC-USDT"}, columns={"price": 65432.1},
                   at=TimestampNanos(event_time_ns))
    df = db.query("SELECT * FROM trades WHERE ts IN '$now-1h..$now'").to_pandas()
```

- One long-lived handle per process; borrow senders from it. Never connect per row.
- `symbols={}` for SYMBOL columns, `columns={}` for the rest, `at=` is
  mandatory. Arrays: `np.float64` numpy arrays, 2D shape `(2, N)` for books.
- Auto-flush every 1,000 rows or 100 ms; `flush(wait=True)` waits for the
  server ack. Reads see writes after WAL apply (`wait_wal_table`).
- Send the event's own timestamp and declare DEDUP keys so replays are
  idempotent.
- `questdb.connect()` takes only `ws::`/`wss::`; `http::`/`tcp::` are the
  legacy ILP `Sender`.

Shell and agents: REST `GET /api/v1/sql/execute?query=...` (use
`curl -G --data-urlencode`), PGWire `psycopg.connect("user=admin password=quest host=localhost port=8812 dbname=qdb")`.

Ports: 9000 (console, REST, QWP, ILP/HTTP), 8812 (PGWire), 9003 (health,
Prometheus), 9009 (legacy ILP/TCP).

```bash
docker run -d --name questdb -p 9000:9000 -p 8812:8812 -p 9003:9003 questdb/questdb:latest
```

---

## Dashboards

Build dashboards as Web Console notebooks through the QuestDB MCP server
(`npx @questdb/mcp-server-questdb@latest setup`). Read
`references/dashboards-mcp.md` before the first call; it covers pairing,
permissions, `apply_notebook_state`, chart configs, variables, time ranges,
per-cell refresh, and two complete reference dashboards (live FX order book,
instance monitoring).

The essentials:
- Pair first (`get_pairing_credentials`, show `userMessage` verbatim, then
  `wait_for_pairing` in the same turn).
- One `apply_notebook_state` call builds the whole grid. It is a full PUT.
- A draw cell can hold several `;`-separated SELECTs overlaid on one chart
  sharing the first query's x-axis; `axis: "right"` with pinned
  `right_axis` min/max keeps different scales readable.
- Set a refresh default; nothing polls otherwise. Fast cells (order book
  200ms-500ms), slow cells (indicators 30s).
- Variables are a `DECLARE` block: reference them as `@symbol`.

---

## Demo Data

`demo.questdb.io` has FX and crypto data. Schema reference:
`https://questdb.com/docs/cookbook/demo-data-schema.md`.

- `fx_trades` (timestamp, symbol, ecn, trade_id, side, passive, price, quantity, counterparty, order_id)
- `market_data` (timestamp, symbol, `bids`/`asks` as `DOUBLE[][]`: `[1]` prices, `[2]` sizes, best_bid, best_ask)
- `core_price` (timestamp, symbol, ecn, bid_price, bid_volume, ask_price, ask_volume, ...)
- Materialized views: `bbo_1s/1m/1h/1d`, `market_data_ohlc_1m/15m/1d`,
  `fx_trades_ohlc_1m/1d`, `core_price_1s/1d`, `trades_OHLC_15m`
- `trades` (crypto: timestamp, symbol, side, price, amount)

`fx_trades.timestamp` is `TIMESTAMP_NS`, so a HORIZON JOIN with `fx_trades`
on the left reports `h.offset` in nanoseconds. Check column types with
`SELECT "column", type FROM table_columns('t')`.
