# QuestDB SQL Grammar Reference

Authoritative index of keywords, functions, data types, and operators available
in QuestDB. Sourced from the sql-parser grammar and verified against live docs.

**Authoritative grammar source:**
https://github.com/questdb/sql-parser/tree/master/src/grammar
(`keywords.ts`, `functions.ts`, `dataTypes.ts`, `operators.ts`, `constants.ts`)

**Ask the server itself.** The connected instance lists every function it
supports, with signatures and kind (`GROUP_BY`, `WINDOW`, `STANDARD`, ...):

```sql
SELECT name, signature_translated, type FROM functions() WHERE name ILIKE '%percentile%';
SELECT * FROM keywords();
```

**When in doubt, fetch the doc page:**
```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/query/functions/finance.md"
curl -s "https://questdb.com/docs/llms.txt"   # full index
```

---

## Notable Keywords

These keywords signal QuestDB-specific features that LLMs commonly miss or
assume don't exist. Standard SQL keywords (SELECT, FROM, WHERE, etc.) are
omitted.

### Time-series query keywords
| Keyword | Feature | Doc path |
|---------|---------|----------|
| `SAMPLE` | Time-bucketed aggregation (`SAMPLE BY 5m`) | `query/sql/sample-by.md` |
| `LATEST` | Most recent row per group (`LATEST ON ts PARTITION BY symbol`) | `query/sql/latest-on.md` |
| `ASOF` | Time-series join: nearest ts <= left | `query/sql/asof-join.md` |
| `LT` | Time-series join: strictly ts < left | `query/sql/join.md` |
| `SPLICE` | Interleave two time series | `query/sql/join.md` |
| `HORIZON` | ASOF at a grid of time offsets in one parallel pass | `query/sql/horizon-join.md` |
| `WINDOW` | Aggregate right-table rows within +- time window | `query/sql/window-join.md` |
| `LATERAL` | Per-outer-row subquery (top-N, dynamic filters) | `query/sql/lateral-join.md` |
| `FILL` | Gap-filling after SAMPLE BY (`FILL(PREV\|PREV(col)\|NULL\|LINEAR\|value)`) | `query/sql/sample-by.md` |
| `ALIGN` | `ALIGN TO CALENDAR` is the default; write it only with `TIME ZONE` / `WITH OFFSET`. `ALIGN TO FIRST OBSERVATION` opts out | `query/sql/sample-by.md` |
| `CUMULATIVE` | Window frame shorthand for running totals | `query/functions/window-functions/overview.md` |
| `SUBSAMPLE` | Reduce rows for charts: `lttb`, `m4`, `minmax`, `uniform`, `cadence`, `sdt` | `query/sql/subsample.md` |
| `GROUPING SETS` / `ROLLUP` / `CUBE` | Subtotals and grand totals with GROUP BY or SAMPLE BY; `GROUPING()`, `GROUPING_ID()` | `query/sql/grouping-sets.md` |
| `DECLARE` | Variables, value lists (`@s := ('A','B')`), `OVERRIDABLE`, `AUDITED` | `query/sql/declare.md` |

### Data reshaping
| Keyword | Feature | Doc path |
|---------|---------|----------|
| `PIVOT` | Reshape rows to columns | `query/sql/pivot.md` |
| `UNNEST` | Expand arrays/JSON arrays into rows | `query/sql/unnest.md` |

### HORIZON/WINDOW JOIN modifiers
| Keyword | Context |
|---------|---------|
| `RANGE` | HORIZON JOIN: `RANGE FROM -1m TO 5m STEP 30s`; WINDOW JOIN: `RANGE BETWEEN 5 seconds PRECEDING AND 5 seconds FOLLOWING` |
| `LIST` | HORIZON JOIN: explicit offsets `LIST (0, 5s, 30s, 1m)` |
| `STEP` | HORIZON JOIN: interval step in RANGE form |
| `PREVAILING` | WINDOW JOIN: `INCLUDE PREVAILING` (default) / `EXCLUDE PREVAILING` |
| `TOLERANCE` | ASOF / LT JOIN staleness bound (`TOLERANCE 50T`) |
| `ANCHOR` | Live view session windows (`ANCHOR DAILY '09:30' 'America/New_York'`) |
| `PRECEDING` | Window/WINDOW JOIN: time or row boundary |
| `FOLLOWING` | Window/WINDOW JOIN: time or row boundary |
| `ORDINALITY` | `UNNEST ... WITH ORDINALITY` adds a 1-based index column |

### DDL / table management
| Keyword | Feature |
|---------|---------|
| `WAL` / `BYPASS WAL` | WAL is the default for partitioned tables; do not write `WAL` |
| `DEDUP` | `DEDUP UPSERT KEYS(...)` (latest wins) or `DEDUP SKIP KEYS(...)` (first wins, 10.2+) |
| `MATERIALIZED` | Materialized views: aggregating (`SAMPLE BY`) or passthrough (no SAMPLE BY) |
| `LIVE` | Live views: `CREATE LIVE VIEW v FLUSH EVERY 1s START FROM NOW AS ...` |
| `EXPIRE ROWS` / `KEEP LATEST` / `KEEP N HIGHEST` | Row expiry on materialized views |
| `REFRESH` | `IMMEDIATE`, `EVERY`, `MANUAL`, `PERIOD (...)`, `DEFERRED`; `REFRESH MATERIALIZED VIEW v FULL` |
| `TTL` | Partition expiry (`TTL 30 DAYS`) |
| `STORAGE POLICY` | Enterprise tiering: `TO PARQUET`, `TO REMOTE`, `DROP LOCAL`, `DROP REMOTE` |
| `FORMAT` | `FORMAT PARQUET` / `NATIVE` partitions; `ALTER TABLE t SET FORMAT ...` |
| `CONVERT PARTITION` | `ALTER TABLE t CONVERT PARTITION TO PARQUET WHERE ...` |
| `PARTITION` | `PARTITION BY HOUR\|DAY\|WEEK\|MONTH\|YEAR\|NONE` |
| `INDEX` | `INDEX`, `INDEX TYPE POSTING INCLUDE (...)`; no CREATE INDEX |
| `DETACH` / `ATTACH` / `DROP PARTITION` | Partition management |
| `SQUASH` | Merge partition splits |
| `RESUME WAL` / `SUSPEND WAL` / `REBASE WAL` | WAL apply control |
| `SHOW CREATE` | `SHOW CREATE TABLE / MATERIALIZED VIEW / LIVE VIEW / DATABASE` |

### Operations
| Keyword | Feature |
|---------|---------|
| `EXPLAIN` | Query plan (`EXPLAIN SELECT ...`) |
| `CANCEL QUERY` | `CANCEL QUERY <id>` from `query_activity()` |
| `TRUNCATE` / `UPDATE` | `TRUNCATE TABLE t`; `UPDATE` works but not on the designated timestamp or Parquet partitions |
| `COPY` | Server-side CSV import `COPY t FROM 'file.csv' WITH HEADER true`; export `COPY (SELECT ...) TO 'dir' WITH FORMAT PARQUET` |
| `CHECKPOINT` / `SNAPSHOT` | `CHECKPOINT CREATE` / `RELEASE` for file-system backups |
| `BACKUP` | Enterprise `BACKUP DATABASE` / `BACKUP ABORT` |
| `VACUUM` | `VACUUM TABLE t` reclaims old column versions |
| `REINDEX` / `FORCE REINDEX PARQUET` | Rebuild indexes (native: `REINDEX TABLE t ... LOCK EXCLUSIVE`; Parquet: `ALTER TABLE t FORCE REINDEX PARQUET`) |
| `ATOMIC` / `BATCH` | `CREATE ATOMIC TABLE ... AS`, `CREATE BATCH n TABLE ... AS` for CTAS |

### Enterprise / auth
| Keyword | Feature |
|---------|---------|
| `GRANT` / `REVOKE` | Permission management |
| `USER` / `USERS` | User management |
| `SERVICE` | Service accounts |
| `TOKEN` | Auth tokens |
| `ASSUME` | Assume service account identity |
| `SWITCH ROLE` | Promote/demote primary and replica in place |
| `RESOURCE GROUP` | Workload isolation (`CREATE RESOURCE GROUP`, `ALTER USER ... SET RESOURCE GROUP`) |
| `WITH AUDIT` | Audited views |

---

## Data Types

Complete list. Use `SYMBOL` for any repeated low-cardinality string (tickers,
categories, status codes) - it is indexed and much faster than VARCHAR.

| Type | Description |
|------|-------------|
| `BOOLEAN` | true/false |
| `BYTE` | 8-bit signed integer |
| `SHORT` | 16-bit signed integer |
| `INT` / `INTEGER` | 32-bit signed integer |
| `LONG` | 64-bit signed integer |
| `LONG128` | 128-bit signed integer |
| `LONG256` | 256-bit unsigned integer |
| `FLOAT` | 32-bit IEEE 754 |
| `DOUBLE` | 64-bit IEEE 754 |
| `DECIMAL` | Arbitrary-precision decimal |
| `CHAR` | Single UTF-16 character |
| `VARCHAR` | Variable-length UTF-8 string |
| `STRING` | Legacy alias for VARCHAR |
| `SYMBOL` | Indexed, interned string (use for repeated values) |
| `TIMESTAMP` | Microsecond-precision UTC timestamp |
| `TIMESTAMP_NS` | Nanosecond-precision UTC timestamp |
| `DATE` | Millisecond-precision date |
| `INTERVAL` | Time interval (pair of timestamps) |
| `UUID` | 128-bit UUID |
| `IPv4` | IPv4 address |
| `GEOHASH` | Variable-precision geohash |
| `BINARY` | Binary data |
| `DOUBLE[]` | 1D double array |
| `DOUBLE[][]` | 2D double array (order books) |
| `FLOAT[]`, `INT[]`, `LONG[]`, `SHORT[]`, `UUID[]` | Typed arrays |

---

## Functions by Category

### Finance / Market Data

Native builtins for order book and execution analysis. **Use these instead of
manual formulas when they fit.**

| Function | Signature | Description |
|----------|-----------|-------------|
| `vwap` | `vwap(price, volume)` | Volume-weighted average price (aggregate). Works on 10.2, not on the docs' finance page, which uses window functions. |
| `twap` | `twap(price, timestamp)` | Time-weighted average price using step-function integration (aggregate). Supports SAMPLE BY + FILL. |
| `mid` | `mid(bid, ask)` | Midpoint of bid and ask. Returns NULL if either is NaN/NULL. |
| `spread` | `spread(bid, ask)` | Bid-ask spread (`ask - bid`) |
| `spread_bps` | `spread_bps(bid, ask)` | Spread in basis points: `spread/mid * 10000` |
| `wmid` | `wmid(bidSize, bidPrice, askPrice, askSize)` | Size-weighted mid (imbalance-adjusted) |
| `l2price` | `l2price(target_size, size_array, price_array)` or `l2price(target_size, size1, price1, size2, price2, ...)` | Average fill price for a market order of target_size against an L2 book |
| `regr_slope` | `regr_slope(y, x)` | Slope of linear regression line. Use for beta, trend. |
| `regr_intercept` | `regr_intercept(y, x)` | Y-intercept of linear regression line |
| `regr_r2` | `regr_r2(y, x)` | Coefficient of determination |

### Aggregation - Basic

| Function | Description |
|----------|-------------|
| `avg(x)` | Arithmetic mean |
| `count()` / `count(col)` | Row count / non-NULL count |
| `count_distinct(col)` | Exact distinct count (use instead of `COUNT(DISTINCT col)`) |
| `sum(x)` | Sum |
| `min(x)` / `max(x)` | Minimum / maximum |
| `first(x)` / `last(x)` | First / last value by designated timestamp |
| `first_not_null(x)` / `last_not_null(x)` | First / last non-NULL value |
| `arg_max(value, key)` / `arg_min(value, key)` | Value at row where key is max/min |
| `ksum(x)` | Kahan compensated sum (better float precision) |
| `nsum(x)` | Neumaier sum (better float precision) |

### Aggregation - Statistical

| Function | Description |
|----------|-------------|
| `stddev(x)` / `stddev_samp(x)` | Sample standard deviation |
| `stddev_pop(x)` | Population standard deviation |
| `var_samp(x)` / `variance(x)` | Sample variance |
| `var_pop(x)` | Population variance |
| `corr(x, y)` | Pearson correlation coefficient |
| `covar_samp(x, y)` | Sample covariance |
| `covar_pop(x, y)` | Population covariance |
| `mode(x)` | Most frequent value |
| `geomean(x)` | Geometric mean (positive values) |
| `weighted_avg(val, weight)` | Weighted mean |
| `weighted_stddev(val, weight)` | Weighted stddev (reliability weights) |
| `weighted_stddev_freq(val, weight)` | Weighted stddev (frequency weights) |
| `weighted_stddev_rel(val, weight)` | Weighted stddev (reliability weights, alias) |
| `percentile_cont(x, p)` / `quantile_cont` | Exact continuous percentile; `p` can be `ARRAY[0.5, 0.95, 0.99]` |
| `percentile_disc(x, p)` / `quantile_disc` | Exact discrete percentile; array form too |
| `kurtosis(x)` / `kurtosis_pop(x)` | Excess kurtosis (sample / population) |
| `skewness(x)` / `skewness_pop(x)` | Skewness (sample / population) |

### Aggregation - Approximate

| Function | Description |
|----------|-------------|
| `approx_count_distinct(col, precision?)` | HyperLogLog distinct count estimate |
| `approx_median(val, precision?)` | Approximate median via HdrHistogram |
| `approx_percentile(val, percentile, precision?)` | Approximate percentile via HdrHistogram |

### Aggregation - String / Boolean / Bitwise

| Function | Description |
|----------|-------------|
| `string_agg(val, delim)` | Concatenate with delimiter |
| `string_distinct_agg(val, delim)` | Concatenate distinct values |
| `bool_and(x)` / `bool_or(x)` | All-true / any-true |
| `bit_and(x)` / `bit_or(x)` / `bit_xor(x)` | Bitwise aggregates |

### Aggregation - Array (element-wise across rows)

| Function | Description |
|----------|-------------|
| `array_elem_avg(arr)` | Element-wise average across arrays |
| `array_elem_sum(arr)` | Element-wise sum |
| `array_elem_min(arr)` / `array_elem_max(arr)` | Element-wise min/max |
| `array_agg(x)` | Collect values (or arrays) into a `DOUBLE[]` |

### Window Functions

All work with `OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE ...)`.

**Frame-respecting** (affected by ROWS BETWEEN / RANGE):
`avg`, `count`, `sum`, `ksum`, `min`, `max`, `first_value`, `last_value`,
`nth_value(x, n)`, `stddev`, `stddev_pop`, `stddev_samp`, `var_pop`,
`var_samp`, `variance`, `corr`, `covar_samp`, `covar_pop`

**Frame-ignoring** (use full partition):
`row_number`, `rank`, `dense_rank`, `percent_rank`, `cume_dist`, `ntile(n)`,
`lag`, `lead`, `percentile_cont/disc(x, p) OVER (PARTITION BY ...)` (no ORDER BY)

**QuestDB extensions:**
- `avg(price, 'period', N) OVER (ORDER BY ts)` - native EMA. Kinds: `'period'`,
  `'alpha'`, or a time constant (`'second'`, `'minute'`, `'hour'`, `'day'`,
  `'week'`). `avg(price, 'period', N, volume)` is VWEMA. Both reject ROWS/RANGE
  frames.
- `sum(x) OVER (ORDER BY ts CUMULATIVE)` - shorthand for running total
- `RANGE BETWEEN '1' HOUR PRECEDING AND CURRENT ROW` - time-based frames
- `WINDOW w AS (...)` named windows, referenced as `OVER w`
- `first_value` / `last_value` / `lag` / `lead` support `IGNORE NULLS` / `RESPECT NULLS`
- SUBSAMPLE keep-flags: `lttb(ts, v, n)`, `m4(ts, v, n)`, `minmax(ts, v, n)`,
  `uniform(n)`, `cadence(n)`, `sdt(ts, v, dev)` `OVER (ORDER BY ts)` return a
  BOOLEAN; filter on it in an outer query

### Array Functions (per-row)

| Function | Description |
|----------|-------------|
| `array_avg(arr)` | Average of elements |
| `array_sum(arr)` | Sum of elements |
| `array_count(arr)` | Count of finite elements |
| `array_min(arr)` / `array_max(arr)` | Min/max element |
| `array_cum_sum(arr)` | Cumulative sum |
| `array_sort(arr, desc?, nullsFirst?)` | Sort elements |
| `array_reverse(arr)` | Reverse order |
| `array_position(arr, elem)` | 1-based index of element |
| `array_build(nArrays, size, filler...)` | Construct array at runtime |
| `array_stddev(arr)` / `array_stddev_pop(arr)` / `array_stddev_samp(arr)` | Stddev of elements |
| `dim_length(arr, dim)` | Length along dimension |
| `dot_product(a, b)` | Dot product of two arrays |
| `matmul(a, b)` | Matrix multiplication (2D) |
| `transpose(arr)` | Transpose (reverse coordinate order) |
| `flatten(arr)` | Flatten to 1D |
| `shift(arr, dist, fill?)` | Shift elements left/right |
| `insertion_point(arr, val, ahead?)` | Binary search insertion point |

### Date-Time Functions

**Current time:**
`now()`, `now_ns()`, `sysdate()`, `systimestamp()`, `systimestamp_ns()`,
`today()`, `tomorrow()`, `yesterday()` (each optionally with timezone arg)

**Extraction:**
`year(ts)`, `month(ts)`, `day(ts)`, `hour(ts)`, `minute(ts)`, `second(ts)`,
`micros(ts)`, `millis(ts)`, `nanos(ts)`, `day_of_week(ts)`,
`day_of_week_sunday_first(ts)`, `week_of_year(ts)`, `days_in_month(ts)`,
`is_leap_year(ts)`, `is_end_of_month(ts)`, `extract(unit, ts)`

**Arithmetic:**
`dateadd(period, n, ts, tz?)`, `datediff(period, ts1, ts2)`,
`timestamp_floor(interval, ts, offset?)`, `timestamp_ceil(unit, ts)`,
`date_trunc(unit, ts)`

**Conversion:**
`to_timestamp(str, fmt)`, `to_timestamp_ns(str, fmt)`, `to_date(str, fmt)`,
`to_str(ts, fmt)` / `to_char`, `to_timezone(ts, tz)`, `to_utc(ts, tz)`,
`to_uuid(lo, hi)`, `to_long128(lo, hi)`, `to_long256(...)`

**Interval:**
`interval(start, end)`, `interval_start(iv)`, `interval_end(iv)`

### String Functions

`concat(...)`, `left(s, n)`, `right(s, n)`, `length(s)`, `length_bytes(s)`,
`substring(s, start, len)`, `split_part(s, delim, idx)`, `position(s, sub)` /
`strpos(s, sub)`, `starts_with(s, prefix)`, `replace(s, old, new)`,
`regexp_replace(s, pattern, replacement)`, `lpad(s, len, fill?)`,
`rpad(s, len, fill?)`, `ltrim(s)`, `rtrim(s)`, `trim(s)`,
`to_lowercase(s)` / `lower(s)`, `to_uppercase(s)` / `upper(s)` / `ucase(s)`,
`quote_ident(s)`

### Numeric Functions

`abs(x)`, `ceil(x)` / `ceiling(x)`, `floor(x)`, `round(x, scale?)`,
`round_up(x, scale)`, `round_down(x, scale)`, `round_half_even(x, scale)`,
`sign(x)`, `sqrt(x)`, `power(x, exp)`, `exp(x)`, `ln(x)`, `log(x)`,
`greatest(...)`, `least(...)`, `size_pretty(bytes)`,
`pi()`, `sin(x)`, `cos(x)`, `tan(x)`, `cot(x)`, `asin(x)`, `acos(x)`,
`atan(x)`, `atan2(y, x)`, `degrees(x)`, `radians(x)`

### Conditional Functions

`CASE WHEN ... THEN ... ELSE ... END`, `coalesce(...)`, `nullif(a, b)`,
`ifnull(x, default)` / `nvl(x, default)`, `isnull(x)`

### JSON Functions

`json_extract(doc, jsonpath)::TYPE` - extract typed values from VARCHAR JSON.
JSONPath syntax: `$.field`, `$.array[0]`, `$.nested.path`.
Cast required for typed output (`::double`, `::varchar`, `::timestamp`, etc.).

### Spatial / Geo Functions

`haversine_dist_deg(lat, lon, ts)` (aggregate: total distance),
`geo_distance_meters(lat1, lon1, lat2, lon2)`,
`geo_within_radius_latlon(lat, lon, clat, clon, radius_m)`,
`within_box(x, y, min_x, min_y, max_x, max_y)`,
`within_radius(x, y, cx, cy, r)`,
`make_geohash(lon, lat, bits)`

### Visualization (inline Unicode charts)

Render data as compact Unicode block charts directly in query results. Output is
`varchar` - works in psql, web console, JDBC, CSV.

Docs: `query/functions/visualization.md`.

| Function | Type | Signature | Description |
|----------|------|-----------|-------------|
| `bar` | Scalar | `bar(value, min, max, width)` | Horizontal bar proportional to value within [min, max]. Width = max characters. Uses `▏▎▍▌▋▊▉█` (8 sub-levels per character). Wraps aggregates: `bar(sum(amount), 0, 50, 30)`. |
| `sparkline` | Aggregate | `sparkline(value)` or `sparkline(value, min, max, width)` | Vertical block chart of values within a group. Uses `▁▂▃▄▅▆▇█`. Each value = one character. Optional `width` sub-samples into buckets. min/max can be NULL for auto-scaling. Pairs with SAMPLE BY. |

### Parquet / External Data

`read_parquet(file_path)` - read a Parquet file as a table. Add
`TIMESTAMP(col)` after it (or after its alias) to use SAMPLE BY / ASOF.
`generate_series(start, end)` - generate a sequence of integers (in `FROM`).

### Hashing / Encoding

`md5(x)`, `sha1(x)`, `sha256(x)`, `base64(x)`

### System / Introspection

These are real functions but rarely needed in analytics queries:

`tables()`, `table_columns(name)`, `table_partitions(name)`,
`table_storage(name)`, `materialized_views()`, `live_views()`, `views()`,
`wal_tables()`, `wal_transactions(name)`, `wait_wal_table(name)`,
`build()`, `version()`, `current_database()`, `session_user()`,
`query_activity()`, `reader_pool()`, `writer_pool()`,
`memory_metrics()`, `table_writer_metrics()`, `node_role()`,
`typeOf(expr)`; Enterprise: `active_permissions()`, `active_grants()`,
`all_permissions()`, `permissions(user)`, `resource_groups()`,
`table_cold_partitions(name)`, `storage_policies`.
Also `functions()`, `keywords()`, `all_tables()`, `current_user()`,
`current_resource_group()`, `backups()` (Enterprise), `import_files()` /
`export_files()` (files under the import/export roots), `sleep(ms)`,
`flush_query_cache()`, `reload_config()`, `reload_tls()`, and PostgreSQL
catalogue compatibility (`pg_class`, `pg_namespace`, `pg_type`,
`current_setting`, ...) for BI tools.
System tables (query by quoted name): `'sys.metrics'`, `'sys.telemetry_wal'`
(see `monitoring.md`).

### Random Data Generation (test data)

`rnd_boolean()`, `rnd_byte(min, max)`, `rnd_short(min, max)`,
`rnd_int(min, max, nanRate)`, `rnd_long(min, max, nanRate)`,
`rnd_float(nanRate)`, `rnd_double(nanRate)`, `rnd_double_array(len, nanRate)`,
`rnd_str(...)`, `rnd_symbol(...)`, `rnd_symbol_weighted(...)`,
`rnd_symbol_zipf(...)` (skewed popularity), `rnd_decimal(...)`, `rnd_log(...)`,
`rnd_interval(...)`,
`rnd_varchar(minLen, maxLen, nanRate)`, `rnd_char()`,
`rnd_date(min, max, nanRate)`, `rnd_timestamp(min, max, nanRate)`,
`rnd_timestamp_ns(min, max, nanRate)`,
`rnd_uuid4()`, `rnd_ipv4(...)`, `rnd_bin(minLen, maxLen, nanRate)`,
`rnd_geohash(bits)`, `rnd_long256()`,
`long_sequence(count)`, `timestamp_sequence(start, step)`,
`timestamp_sequence_ns(start, step)`, `timestamp_shuffle(ts1, ts2)`

---

## Operators

Standard SQL operators plus:

| Operator | Description |
|----------|-------------|
| `~` | Regex match |
| `!~` | Regex not match |
| `~=` | Regex match (case insensitive) |
| `ILIKE` | Case-insensitive LIKE |
| `IN` | Set membership / TICK interval syntax |
| `BETWEEN` | Range check |
| `IS [NOT] NULL` | NULL check |
| `PIVOT` | In the FROM clause (no UNPIVOT keyword: use UNION ALL) |

All doc paths are relative to `https://questdb.com/docs/`.
