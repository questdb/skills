# Advanced Patterns: Bulk Export, Backfill, Replay, Query Shape

Patterns taken from working QuestDB 10.x projects
(`questdb/blog-examples/2026-11-market-data-replay` and similar). Python
shown; the same shapes apply to every QWP client.

## Streaming Results

```python
with questdb.connect("ws::addr=localhost:9000;") as db, db.reader() as reader:
    with reader.query(sql, reset_symbol_dict=False) as result:
        for batch in result.iter_arrow():          # also iter_pandas(), iter_polars()
            if batch.num_rows == 0:                # a result can open with a schema-only batch
                continue
            consume(batch)
            if done:
                result.cancel()                    # stop the server early
```

- `to_pandas()` / `to_polars()` materialise the whole result; `iter_*()`
  streams batches (up to 16,384 rows) with flat memory. A slow consumer slows
  the server down through QWP flow control.
- A reader lease and each `QueryResult` belong to the thread that created
  them. One connection runs one query at a time: parallelism means one
  connection (and lease) per thread.
- `reset_symbol_dict=False` keeps the SYMBOL dictionary warm across queries on
  the same lease that read the same columns.
- If a lease raises an error, close it and take a fresh `db.reader()`; a
  terminal error poisons the lease for its next call. Reader failover itself is
  on by default.
- `pl.from_arrow(batch)` is zero-copy; read with one lease and write back with
  `db.dataframe(...)` on the same handle (separate pooled connections).

## Fastest Full Export: Split by the Data Distribution

Equal time spans are skewed (busy hours hold most rows). Ask QuestDB for the
distribution first, then cut slices with roughly equal row counts and stream
them over N connections in parallel.

```python
import collections, threading
import questdb

TABLE, TS = "market_data", "timestamp"
WHERE = " WHERE timestamp IN '$yesterday'"
N = 8

with questdb.connect(CONF) as db:
    m = db.query(f"SELECT {TS} minute, count() rows FROM {TABLE}{WHERE} SAMPLE BY 1m").to_pandas()

# Bucket starts as UTC literals (minute-aligned, so no fractional seconds to lose)
ts = [t.strftime("%Y-%m-%dT%H:%M:%S") + "Z" for t in m["minute"]]
counts = m["rows"].tolist()
target = max(-(-sum(counts) // (N * 4)), 1)         # a few slices per connection, finishing together
slices, lo, rows = collections.deque(), ts[0], 0
for minute, count in zip(ts, counts):
    if rows >= target:
        slices.append((lo, minute)); lo, rows = minute, 0
    rows += count
slices.append((lo, None))                           # last slice open-ended

def worker():
    with questdb.connect(CONF) as db, db.reader() as reader:
        while True:
            try:
                lo, hi = slices.popleft()
            except IndexError:
                return
            cond = f"{TS} >= '{lo}'" + (f" AND {TS} < '{hi}'" if hi else "")
            with reader.query(f"SELECT * FROM {TABLE}{WHERE} AND {cond}", reset_symbol_dict=False) as r:
                for batch in r.iter_arrow():
                    if batch.num_rows:
                        publish(batch)

threads = [threading.Thread(target=worker) for _ in range(N)]
for t in threads: t.start()
for t in threads: t.join()
```

- Slices start and end on bucket boundaries (half-open `[lo, hi)`), so no row
  is read twice. Keep the original filters (`symbol`, time range) in every
  slice query.
- Cap rows per slice so each query finishes within `query.timeout`
  (60 s default).
- Rows are ordered within a slice, not across slices. For one globally
  ordered stream, give each worker a contiguous range and a bounded
  `queue.Queue(maxsize=8)`, then drain queue 0, then 1, and so on: peak memory
  is `workers × 8` batches.
- Bounds that are not bucket-aligned on nanosecond tables need 9 fractional
  digits in the literal.
- Throughput per connection is capped by socket buffers on default Linux
  (about 416 KB); several connections, or raising `net.core.wmem_max` /
  `rmem_max`, lift it.
- Alternative for static tables: `LIMIT -m, -n` row-offset chunks need no
  distribution query, but offsets shift under concurrent appends.

## Parallel Backfill

- Sort each batch on the designated timestamp before sending, and give each
  process a **contiguous** time range: QuestDB then appends instead of merging
  out-of-order data.
- One process per worker, each with its own `sender_id` (and its own `sf_dir`
  with store-and-forward; the client does not create the directory).
- Backpressure: poll
  `SELECT sequencerTxn, writerTxn FROM wal_tables() WHERE name = $1` and pause
  the writers while `sequencerTxn - writerTxn` grows (e.g. beyond
  `3 × workers`). More than about three writers into one table mostly buys
  merge work, not speed.
- Merging staging tables into a final one: one time window at a time,
  `INSERT INTO final SELECT * FROM (... UNION ALL ...) ORDER BY ts`. Run
  long DDL/DML over REST with a `Statement-Timeout` header.
- Before reading back, wait for `writerTxn = sequencerTxn` (and stop if the
  table is `suspended`), or call `wait_wal_table('t')`.

## Real-Time Generators and Replays

- Write small slices (e.g. 250 ms) slightly ahead of the wall clock; when
  more than a few seconds behind, catch up in larger chunks, then sleep until
  the next slice is due.
- Never write before the latest timestamp already in the table; persist
  generator state so a restart continues without jumps.
- Use the event's own timestamp. `at_now()` / server timestamps stamp a whole
  batch with one value and collapse rows under DEDUP on the timestamp.
- Corrections as new rows with `version + 1` and
  `DEDUP UPSERT KEYS(ts, symbol, version)`; keep both event time and booking
  time for bitemporal queries.

## Query Shape Tips

- **HORIZON JOIN: filter the right side inside it.** A `WHERE` after the join
  filters only the left table; put the right side's time range in the
  subquery so it does not scan every quote:

  ```sql
  DECLARE @day := '2026-10-13'
  SELECT h.offset / 1000000 AS horizon_s, t.book, count() AS fills,
         avg(10000 * CASE WHEN t.qty > 0 THEN 1 ELSE -1 END * (mid(q.bid, q.ask) - t.px) / t.px) AS markout_bps
  FROM (fills WHERE ts IN @day) AS t
  HORIZON JOIN (quotes WHERE ts >= @day::timestamp
                        AND ts < dateadd('m', 16, dateadd('d', 1, @day::timestamp))) AS q
    ON (t.symbol = q.symbol AND t.venue = q.source)
  LIST (0, 1s, 10s, 1m, 5m, 15m) AS h
  ORDER BY t.book, horizon_s;
  ```

  The right range extends past the left one by the largest offset.
- **ASOF JOIN against a filtered subquery**, `ASOF JOIN (quotes WHERE symbol = @s) q TOLERANCE 30s`,
  rather than against a CTE: the subquery form takes the fast path.
- **LATEST ON then filter**: a `WHERE` at the same level runs before
  LATEST ON. To get the latest row and then keep only some statuses, filter in
  an outer query.
- **"As of" views**: a parameter with a default, overridden per query:

  ```sql
  CREATE OR REPLACE VIEW marks_asof AS (
    DECLARE OVERRIDABLE @asof := now()
    SELECT symbol, price, version, ts FROM curve_marks
    WHERE ts <= @asof AND ts > dateadd('d', -1, @asof::timestamp)
    LATEST ON ts PARTITION BY symbol
  );
  DECLARE @asof := '2026-10-13T16:00:00Z' SELECT * FROM marks_asof;
  ```
- **Force a designated timestamp** on a CTE or subquery so SAMPLE BY and
  time joins work: `FROM (cte ORDER BY ts) TIMESTAMP(ts)`.
- Per-column Parquet encodings for heavy tables:
  `ts TIMESTAMP PARQUET(delta_binary_packed, zstd(4))`,
  `symbol SYMBOL PARQUET(rle_dictionary, zstd(4), bloom_filter)`.
