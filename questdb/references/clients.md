# Connecting to QuestDB: QWP Clients, REST, PGWire, ILP

Since QuestDB 10.0 every native client speaks **QWP** (QuestDB Wire Protocol), a
binary protocol over WebSocket on the HTTP port (9000). One connection handle
does both **ingestion** and **SQL queries**, pools senders and readers, and
supports multi-host failover and store-and-forward. Prefer QWP for all new code.

| Need | Use |
|------|-----|
| Ingest from an app or pipeline | QWP client (`ws::` / `wss::` connect string) |
| Query from an app (sustained or large results) | QWP client (same handle) |
| Quick schema discovery, small result sets, shell scripts | REST `/api/v1/sql/execute` |
| BI tools, psql, Grafana | PGWire on 8812 |
| Existing Telegraf / InfluxDB / Flink pipelines that emit ILP | ILP (compatibility protocol) |

Docs: `https://questdb.com/docs/connect/overview.md`,
`https://questdb.com/docs/connect/clients/connect-string.md`

---

## Connect String

```
ws::addr=localhost:9000;                                       # OSS, local
wss::addr=questdb.example.com:9000;username=admin;password=quest;
wss::addr=node-a:9000,node-b:9000;token=YOUR_TOKEN;            # Enterprise, failover
wss::addr=node-a:9000,node-b:9000;sf_dir=/var/lib/app/qdb-sf;sender_id=ingest-1;
```

- Schemes: `ws` and `wss`. Both default to port **9000** (`wss` does NOT default to 443).
- No `protocol_version` key: QWP negotiates its version during the WebSocket upgrade.
- Auth: `username`/`password` (aliases `user`/`pass`), or `token` (bearer,
  recommended on Enterprise). Never hardcode tokens; read them from the
  environment. Clients also read the whole string from `QDB_CLIENT_CONF`.
- TLS: `tls_roots`, `tls_ca=os_roots`; `tls_verify=unsafe_off` only for
  self-signed dev certificates, and only after asking the user.
- **Failover (Enterprise):** list several hosts, comma-separated or repeated
  `addr=`. OSS is single-node. A running sender retries outages indefinitely;
  the first connect fails fast unless `initial_connect_retry=on` or a
  `reconnect_*` key is set. Queries fail over by default (`failover=on`).
- **Store-and-forward:** off by default (memory-only buffering). Set `sf_dir`
  to buffer to disk and replay after a restart.
- Tables behind multi-host failover **must** declare DEDUP keys, because replay
  can resend rows.
- Auto-flush defaults (Python): 1,000 rows or 100 ms, checked on `row()`, no
  background timer. Keys: `auto_flush_rows`, `auto_flush_interval`,
  `auto_flush=off`.

### Exactly-once

Clients deliver **at-least-once**. For exactly-once outcomes:
1. Send the event's own timestamp in `at=`. Server-assigned timestamps
   (`at_now()`, `ServerTimestamp`, `TimestampNanos.now()`) differ on replay.
2. Declare `DEDUP UPSERT KEYS(ts, ...)` or `DEDUP SKIP KEYS(ts, ...)` covering
   row identity.
3. Keep values stable across retries.

---

## Python (`pip install -U "questdb[dataframe]"`, Python 3.10+)

The plain `questdb` package only needs numpy; `to_pandas()` and `db.dataframe()`
raise `ModuleNotFoundError: No module named 'pandas'` without the
`dataframe` extra (pandas + pyarrow). Use `questdb[polars]` for polars.

```python
import questdb
from questdb import QuestDBError, TimestampNanos

with questdb.connect("ws::addr=localhost:9000;") as db:
    # DDL / DML
    db.execute("""
        CREATE TABLE IF NOT EXISTS trades (
            ts TIMESTAMP, symbol SYMBOL, side SYMBOL, price DOUBLE, amount DOUBLE
        ) TIMESTAMP(ts) PARTITION BY DAY
        DEDUP UPSERT KEYS(ts, symbol)
    """)

    # Ingest rows
    with db.sender() as sender:
        sender.row(
            "trades",
            symbols={"symbol": "BTC-USDT", "side": "buy"},   # SYMBOL columns
            columns={"price": 65432.10, "amount": 0.0012},    # everything else
            at=TimestampNanos(event_ts_nanos),                # designated timestamp, mandatory
        )
        sender.flush(wait=True)   # wait=True blocks until the server acks

    # Query
    with db.query("SELECT * FROM trades WHERE ts IN '$now-1h..$now'") as result:
        df = result.to_pandas()          # also to_polars(), to_arrow(), iter_*()

    # Bind parameters
    df = db.query("SELECT * FROM trades WHERE symbol = $1 AND price > $2",
                  ["ETH-USDT", 2615.0]).to_pandas()
```

Rules:
- `questdb.connect()` accepts only `ws`/`wss` strings. `http::`/`tcp::` belong
  to the legacy ILP `Sender`. `questdb.ingress` is a deprecated alias module.
- `row()` needs `at=`: a `TimestampNanos`, a `datetime`, or
  `questdb.ServerTimestamp`. When QWP auto-creates a table, its designated
  timestamp column is named `timestamp` and typed `TIMESTAMP_NS`. Create the
  table first if you want another name or type, DEDUP keys or a partition
  interval.
- Python `float` becomes DOUBLE, `int` becomes LONG, `str` in `columns` becomes
  VARCHAR, numpy `float64` arrays become `DOUBLE[]` / `DOUBLE[][]`. Nulls are
  written by omitting the column.
- `flush()` returns before the server acks; `flush(wait=True)` waits. A `with`
  block that exits normally flushes; one that exits on an exception drops the
  rows still buffered in that lease.
- Query visibility after a write is asynchronous (WAL apply). Poll, or use
  `SELECT wait_wal_table('trades')` before a read that must see the rows.
- Use one long-lived `db` handle per process. Do not open a connection per row
  or per callback.

**2D arrays (order books):** shape `(2, N)`, row 0 = prices, row 1 = sizes,
`dtype=np.float64` only.

```python
import numpy as np
bids = np.array([[64901.6, 64901.5], [3.02, 0.06]], dtype=np.float64)
asks = np.array([[64901.7, 64901.8], [1.54, 0.21]], dtype=np.float64)
sender.row("order_book", symbols={"symbol": "BTC-USDT"},
           columns={"bids": bids, "asks": asks}, at=TimestampNanos.now())
```

**DataFrames** (pandas, polars DataFrame/LazyFrame, pyarrow Table/RecordBatch):

```python
db.dataframe(df, table_name="trades", symbols=["symbol"], at="timestamp")
```

`dataframe()` is not atomic and blocks for an ack; a retry duplicates rows
unless the table has DEDUP keys. It bypasses store-and-forward. pandas
columns meant as SYMBOL should be `Categorical` (or listed in `symbols=`);
`object` dtype lands as VARCHAR.

**Streaming large results and parallel reads** (`iter_arrow()`, one reader
lease per thread, slicing by the data distribution): see `patterns.md`.

Errors: `QuestDBError` with `e.code` and `e.in_doubt`.

---

## Other Languages

All follow the same shape: connect once, borrow a sender or a query lease.

**Java** (`org.questdb:questdb-client`)
```java
try (QuestDB db = QuestDB.connect("ws::addr=localhost:9000;")) {
    try (Sender sender = db.borrowSender()) {
        sender.table("trades")
              .symbol("symbol", "ETH-USD")
              .doubleColumn("price", 2615.54)
              .doubleColumn("amount", 0.00044)
              .at(Instant.now());
    }
}
// Query: db.borrowQuery(), q.sql(...).handler(new QwpColumnBatchHandler(){...}).submit()
// Arrays: sender.table("book").doubleArray("levels", prices).atNow();
```

**Go** (`github.com/questdb/go-questdb-client/v4`, pooled API is beta: `go get ...@main`, Go 1.23+)
```go
db, err := qdb.Connect(ctx, "ws::addr=localhost:9000;")
sender, err := db.BorrowSender(ctx)
err = sender.Table("trades").Symbol("symbol", "ETH-USD").
	Float64Column("price", 2615.54).AtNow(ctx)
// Query: q, _ := db.BorrowQuery(ctx); cursor := q.Query(ctx, sql)
//        for batch, err := range cursor.Batches() { ... }
```
`Flush` returning `nil` does not mean the server accepted the rows.

**Rust** (`questdb-rs = "7"`)
```rust
use questdb::{ingress::TimestampNanos, QuestDb};
let db = QuestDb::connect("ws::addr=localhost:9000;")?;
let mut sender = db.borrow_sender()?;
let mut buffer = sender.new_buffer();
buffer.table("trades")?.symbol("symbol", "ETH-USD")?
      .column_f64("price", 2615.54)?.at(TimestampNanos::now())?;
sender.flush_buffer(&mut buffer)?;
// Query: db.borrow_reader()?.prepare(sql).execute()?, cursor.next_batch()?
```
Rust has no auto-flush.

**C/C++** (`c-questdb-client`, CMake FetchContent, needs a Rust toolchain)
```cpp
questdb::pool pool{"ws::addr=localhost:9000;"};
auto sender = pool.borrow_sender();
auto buffer = sender.new_buffer();
buffer.table("trades"_tn).symbol("symbol"_cn, "ETH-USD"_utf8)
      .column("price"_cn, 2615.54).at(questdb::ingress::timestamp_nanos::now());
sender.flush_and_wait(buffer);
```

**.NET** (`dotnet add package net-questdb-client`, .NET 7+)
```csharp
await using var client = QuestDBClient.Connect("ws::addr=localhost:9000;");
using (var sender = client.BorrowSender()) {
    await sender.Table("trades").Symbol("symbol", "ETH-USD")
        .Column("price", 2615.54).AtAsync(DateTime.UtcNow);
    await sender.SendAsync();   // dispose does NOT send; unsent rows are discarded
}
await using var reader = await client.ExecuteReaderAsync("SELECT count() FROM trades");
```

**Node.js** (`@questdb/nodejs-client` 5.0+, Node 20.18.1+)
```typescript
import { connectQwpNodeClient } from "@questdb/nodejs-client";
const db = await connectQwpNodeClient("ws::addr=localhost:9000;");
const sender = await db.borrowSender();
await sender.table("trades").symbol("symbol", "ETH-USD")
  .floatColumn("price", 2615.54).at(Date.now(), "ms");   // at() without unit = microseconds
await sender.flushAndWait(10_000);
await sender.close();
// Query: const lease = await db.borrowQuery(); for await (const batch of await lease.query(sql)) ...
```
LONG and TIMESTAMP values come back as `bigint`.

**Ruby** (`gem "questdb", "~> 0.1.0"`, MRI 3.1-3.4)
```ruby
QuestDB.connect("ws::addr=localhost:9000;") do |client|
  client.sender do |sender|
    sender.row("trades", symbols: { symbol: "ETH-USD" },
               columns: { price: 2615.54 }, at: QuestDB::TimestampNanos.now)
    sender.flush(wait: true)
  end
  rows = client.query("SELECT * FROM trades WHERE symbol = $1", binds: ["ETH-USD"]).to_a
end
```

**Browser** (`@questdb/browser-client`, QuestDB 10.1+). Takes an options object,
not a connect string. Same `borrowSender()` / `borrowQuery()` API as Node.js.
```typescript
import { connectQwpBrowserClient } from "@questdb/browser-client";
const origin = new URL("/", window.location.href);
origin.protocol = origin.protocol === "https:" ? "wss:" : "ws:";
const db = await connectQwpBrowserClient({ cluster: { url: origin } });
```
Serve the page from the same origin (reverse proxy) or list its origin in
`qwp.browser.allowed.origins`. Unacked rows live only in page memory.

Full pages: `https://questdb.com/docs/connect/clients/{python,java,go,rust,c-and-cpp,dotnet,nodejs,ruby,browser}.md`

---

## REST API

```bash
curl -G --data-urlencode "query=SELECT * FROM trades LIMIT 5" \
  http://localhost:9000/api/v1/sql/execute
```

- `/api/v1/sql/execute` (short `/exec` still works but is deprecated).
  **GET only**; POST returns 405.
- Params: `query` (URL-encoded), `count`, `limit` (`10` or `10,20`), `nm`,
  `timings`, `explain`. Header `Statement-Timeout` (ms). No bind parameters.
- Response: `{"query", "columns":[{name,type}], "timestamp": <index or -1>, "dataset":[[...]], "count"}`.
  DDL returns `{"ddl":"OK"}`.
- `/api/v1/sql/validate` compiles without running and returns column metadata.
- Auth: basic auth on OSS (`http.user`/`http.password`) and Enterprise;
  `Authorization: Bearer <token>` on Enterprise.
- Use `--data-urlencode` rather than hand-encoding the query.
- Other endpoints: `/imp` (POST multipart CSV import), `/exp` (GET, CSV or Parquet export).

## PGWire

```python
import psycopg   # psycopg v3, not psycopg2
conn = psycopg.connect("user=admin password=quest host=localhost port=8812 dbname=qdb")
```

Database name is ignored (`qdb` by convention). Defaults `admin`/`quest`. No
SSL, no `COPY FROM STDIN`, no `DELETE`.

## ILP (compatibility protocol)

ILP remains fully supported for tools that already emit it (Telegraf, Kafka
pipelines, InfluxDB migrations). New code should use QWP. Transports: HTTP on
9000 (preferred, returns errors) and TCP on 9009 (legacy, errors are not
reported back). In Python use the standalone `questdb.Sender` with
`http::addr=localhost:9000;` or `tcp::addr=localhost:9009;protocol_version=2;`
(protocol 2 is needed for arrays).

## Ports

| Port | Service |
|------|---------|
| 9000 | Web Console, REST, QWP (ingest + query), ILP/HTTP |
| 8812 | PGWire |
| 9003 | Health / metrics (`/metrics` Prometheus) |
| 9009 | ILP/TCP (legacy) |

```bash
docker run -d --name questdb -p 9000:9000 -p 8812:8812 -p 9003:9003 questdb/questdb:latest
```
