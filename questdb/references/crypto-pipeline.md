# Real-Time Crypto Pipeline (cryptofeed + QuestDB)

A ready template: OKX trades and L2 books into QuestDB over QWP, candle
materialized views, and a live dashboard. **Copy it as-is.** Do not explore
the cryptofeed or questdb package sources, check versions, or verify
infrastructure the user said is running.

Files: `01_schema.py` (tables and views), `02_ingest.py` (feed handler),
`03_start.py` (launches ingestion detached and waits for rows). The
dashboard is built with the QuestDB MCP server (`dashboards-mcp.md`), or with
Grafana only if the user asks (`grafana.md`).

## Environment

QuestDB, if not running:
```bash
docker run -d --name questdb -p 9000:9000 -p 8812:8812 -p 9003:9003 questdb/questdb:latest
```
```bash
curl -sf --retry 30 --retry-delay 1 --retry-all-errors -o /dev/null "http://localhost:9000/api/v1/sql/execute?query=SELECT+1"
```

Fresh venv (each command its own Bash call):
```bash
python3 -m venv .venv
```
```bash
.venv/bin/pip install -q -U cryptofeed "questdb[dataframe]" numpy certifi
```
```bash
.venv/bin/pip uninstall uvloop -y
```

- **uvloop:** a fresh venv may pull a uvloop that crashes on Python 3.10+
  (`RuntimeError: no current event loop`). Uninstall it in fresh venvs only;
  never touch a venv the user provided.
- **macOS SSL:** Homebrew Python lacks CA certificates. Set `SSL_CERT_FILE`
  from certifi **inside Python** at the top of the script, never as a shell
  prefix.
- The questdb client needs Python 3.10+.

## 01_schema.py

```python
import questdb

DDL = [
    """CREATE TABLE IF NOT EXISTS trades (
        ts TIMESTAMP, symbol SYMBOL, side SYMBOL, price DOUBLE, amount DOUBLE
    ) TIMESTAMP(ts) PARTITION BY DAY TTL 7 DAYS
    DEDUP UPSERT KEYS(ts, symbol)""",
    """CREATE TABLE IF NOT EXISTS orderbook (
        ts TIMESTAMP, symbol SYMBOL, bids DOUBLE[][], asks DOUBLE[][]
    ) TIMESTAMP(ts) PARTITION BY HOUR TTL 2 DAYS
    DEDUP UPSERT KEYS(ts, symbol)""",
    """CREATE MATERIALIZED VIEW IF NOT EXISTS candles_1s AS (
        SELECT ts, symbol, first(price) AS open, max(price) AS high,
               min(price) AS low, last(price) AS close, sum(amount) AS volume
        FROM trades SAMPLE BY 1s
    ) PARTITION BY DAY TTL 7 DAYS""",
    """CREATE MATERIALIZED VIEW IF NOT EXISTS candles_1m AS (
        SELECT ts, symbol, first(open) AS open, max(high) AS high,
               min(low) AS low, last(close) AS close, sum(volume) AS volume
        FROM candles_1s SAMPLE BY 1m
    ) PARTITION BY DAY TTL 30 DAYS""",
    """CREATE MATERIALIZED VIEW IF NOT EXISTS candles_1h AS (
        SELECT ts, symbol, first(open) AS open, max(high) AS high,
               min(low) AS low, last(close) AS close, sum(volume) AS volume
        FROM candles_1m SAMPLE BY 1h
    ) PARTITION BY MONTH""",
]

with questdb.connect("ws::addr=localhost:9000;") as db:
    for stmt in DDL:
        db.execute(stmt)
print("schema ready")
```

## 02_ingest.py

```python
import os
import certifi
os.environ['SSL_CERT_FILE'] = certifi.where()  # macOS: Homebrew Python lacks CA certs

import numpy as np
import questdb
from questdb import TimestampNanos
from cryptofeed import FeedHandler
from cryptofeed.exchanges import OKX
from cryptofeed.defines import TRADES, L2_BOOK

CONF = "ws::addr=localhost:9000;"   # Enterprise: see references/enterprise.md
SYMBOLS = ['BTC-USDT', 'ETH-USDT', 'SOL-USDT']
DEPTH = 25

sender = None   # one long-lived lease, auto-flushes every 1,000 rows or 100 ms

async def trade_cb(t, receipt_timestamp):
    sender.row(
        'trades',
        symbols={'symbol': t.symbol, 'side': t.side},
        columns={'price': float(t.price), 'amount': float(t.amount)},
        at=TimestampNanos(int(t.timestamp * 1e9)),
    )

async def book_cb(book, receipt_timestamp):
    bids, asks = book.book['bid'], book.book['ask']
    bid_px = sorted(bids.keys(), reverse=True)[:DEPTH]   # best first
    ask_px = sorted(asks.keys())[:DEPTH]
    sender.row(
        'orderbook',
        symbols={'symbol': book.symbol},
        columns={
            # shape (2, N): row 1 = prices, row 2 = sizes (bids[1][1] = best bid)
            'bids': np.array([[float(p) for p in bid_px], [float(bids[p]) for p in bid_px]], dtype=np.float64),
            'asks': np.array([[float(p) for p in ask_px], [float(asks[p]) for p in ask_px]], dtype=np.float64),
        },
        at=TimestampNanos(int(book.timestamp * 1e9)),
    )

with questdb.connect(CONF) as db:
    with db.sender() as s:
        sender = s
        f = FeedHandler()
        f.add_feed(OKX(symbols=SYMBOLS, channels=[TRADES, L2_BOOK],
                       callbacks={TRADES: trade_cb, L2_BOOK: book_cb}))
        f.run()
```

## 03_start.py (launch detached and wait for data)

Shell redirection and `&` trigger permission prompts in some agents, so launch
from Python:

```python
import os, subprocess, sys, time
import questdb

HERE = os.path.dirname(os.path.abspath(__file__))
log = open(os.path.join(HERE, "ingest.log"), "a")
proc = subprocess.Popen([sys.executable, "-u", os.path.join(HERE, "02_ingest.py")],
                        stdout=log, stderr=log, cwd=HERE,      # cryptofeed writes feedhandler.log to cwd
                        start_new_session=True)                # survives this script
print(f"ingestion PID {proc.pid}")

with questdb.connect("ws::addr=localhost:9000;") as db:
    for i in range(20):
        time.sleep(1)
        if proc.poll() is not None:
            sys.exit("ingestion died, see ingest.log")
        n = db.query("SELECT count() FROM trades").to_pandas().iloc[0, 0]
        if n > 0:
            print(f"{n} rows after {i + 1}s")
            break
    else:
        sys.exit("no data after 20s, see ingest.log")
```

Run `01_schema.py`, then `03_start.py`. Do not start `02_ingest.py` by hand,
and never restart or kill it unless the user asks.

**cryptofeed API (tested with 2.4.x and 2.5.0, do not read its source):**
- Trades: `t.symbol`, `t.side` (`'buy'`/`'sell'`), `t.price`, `t.amount`
  (`Decimal`, cast with `float()`), `t.timestamp` (float epoch seconds).
- Books: `book.book['bid']` / `['ask']` are `{Decimal(price): Decimal(size)}`;
  `book.symbol`, `book.timestamp`.
- Exchanges: `OKX`, `Coinbase`, `Binance`, `Kraken`, `Bybit`, ... Symbols are
  exchange-native (`BTC-USDT` on OKX, `BTC-USD` on Coinbase).
- Channels: `TRADES`, `L2_BOOK`, `L3_BOOK`, `TICKER`, `CANDLES`,
  `OPEN_INTEREST`, `FUNDING`, `LIQUIDATIONS`.
- cryptofeed logs to stderr and to `feedhandler.log` in its working
  directory; empty stdout is normal. OKX "port 8443 will be retired" notices
  in the log are harmless.
- Avoid `X | None` type hints in scripts that may run on older Pythons.

## Dashboard (Web Console via MCP)

Pair the MCP server, then create one notebook with `apply_notebook_state`
(`layout_mode: "grid"`, `auto_refresh_default: "1s"`, variables
`symbol := 'BTC-USDT'`). Suggested cells:

| Cell | SQL source | Chart | Refresh |
|------|------------|-------|---------|
| OHLC + volume, last 5 min | `candles_1s` | candlestick + bar on right axis | `1s` |
| VWAP, Bollinger, RSI on the candles | CTEs over `candles_1s` | candlestick + lines, RSI on right axis 0..100 | `5s` |
| Depth (pips or price from mid) | `orderbook LATEST ON ts PARTITION BY symbol` + `UNNEST` | stepArea, numeric x | `500ms` |
| Mid and spread | `orderbook SAMPLE BY 1s` | mid line left, spread in bps on right axis (best bid and ask lines overlap at one tick) | `1s` |
| Last trades table | `trades WHERE symbol = @symbol ORDER BY ts DESC LIMIT 20` | table, highlight `side`, `steps` on notional | `500ms` |

Depth for crypto, in bps from mid (pips do not apply). 25 OKX levels span
about 2 bps on BTC-USDT, 3 on ETH-USDT and 28 on SOL-USDT, so the constant
width edge is set per symbol:

```sql
WITH book AS (
  SELECT bids, asks FROM orderbook
  WHERE symbol = @symbol AND ts IN '$now-1m..$now'
  LATEST ON ts PARTITION BY symbol
), m AS (
  SELECT (bids[1][1] + asks[1][1]) / 2 AS mid,
         CASE WHEN @symbol = 'BTC-USDT' THEN 2.5 WHEN @symbol = 'ETH-USDT' THEN 4 ELSE 30 END AS edge
  FROM book
), b AS (
  SELECT round(10000 * (u.price - m.mid) / m.mid, 3) AS bps, u.depth, m.edge
  FROM book b CROSS JOIN m, UNNEST(b.bids[1], array_cum_sum(b.bids[2])) u(price, depth)
), a AS (
  SELECT round(10000 * (u.price - m.mid) / m.mid, 3) AS bps, u.depth, m.edge
  FROM book b CROSS JOIN m, UNNEST(b.asks[1], array_cum_sum(b.asks[2])) u(price, depth)
)
SELECT bps AS bps_from_mid, depth AS bid_depth, NULL::double AS ask_depth FROM b WHERE bps >= -edge
UNION ALL SELECT bps, NULL::double, depth FROM a WHERE bps <= edge
UNION ALL SELECT -first(edge), max(depth), NULL::double FROM b WHERE bps >= -edge
UNION ALL SELECT first(edge), NULL::double, max(depth) FROM a WHERE bps <= edge
ORDER BY bps_from_mid
```
`chart_config`: `x_column: "bps_from_mid"`, one `stepArea` query with
`y_columns: ["bid_depth", "ask_depth"]`.

Filters: `WHERE symbol = @symbol AND ts IN '$now-5m..$now'`. The depth and
overlay patterns are in `dashboards-mcp.md` and
`examples/fx-live-dashboard.json`; adapt `market_data`/`timestamp` to
`orderbook`/`ts`.

Definition of done: ingestion still running, and the notebook exists with
every statement reporting `success`. Tell the user the tab name and offer to
switch to it.
