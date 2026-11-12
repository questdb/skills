# Financial Indicator SQL Recipes

Tested SQL from the QuestDB cookbook. OHLC, VWAP, Bollinger Bands and RSI are
in `examples/fx-live-dashboard.json` (notebook form) and `grafana.md` (Grafana
form); this file covers the others.

## Column Mapping

Demo tables use different schemas from the user's actual tables. Use this mapping:

| Demo column | Trades table (`trades`) | OHLC table (user-built via SAMPLE BY) |
|---|---|---|
| `timestamp` | `ts` | `ts` |
| `close` | N/A: use `last(price)` in SAMPLE BY | `close` |
| `open` | N/A: use `first(price)` in SAMPLE BY | `open` |
| `high` | N/A: use `max(price)` in SAMPLE BY | `high` |
| `low` | N/A: use `min(price)` in SAMPLE BY | `low` |
| `total_volume` / `volume` | `sum(amount)` in SAMPLE BY | `volume` |
| `quantity` | `amount` | N/A |
| `price` | `price` | N/A |

**For Grafana panels** (only when the user asks for Grafana): use the
Grafana-ready versions in the next section. They use `$__timeFilter`,
`$symbol`, and `$__interval` with the `trades` table directly.

Demo tables used in standalone recipes below:
- `market_data_ohlc_15m`: pre-aggregated 15m OHLC candles (timestamp, symbol, open, high, low, close)
- `fx_trades`: raw tick trades (timestamp, symbol, price, quantity)
- `fx_trades_ohlc_1m`: materialized view 1m OHLC (timestamp, symbol, open, high, low, close, total_volume)
- `core_price`: bid/ask quotes (timestamp, symbol, bid_price, ask_price)
- `market_data`: order book with arrays (timestamp, symbol, bids, asks)
- `trades`: crypto trades (timestamp, symbol, price, amount, side)

Key QuestDB pattern: `avg(value, 'period', N)` is the native EMA window function
(α = 2/(N+1)). For Wilder's smoothing (α = 1/N), use `avg(value, 'period', 2N-1)`
or `avg(value, 'alpha', 1.0/N)`. EMA works on CTE-computed columns, and rejects
ROWS/RANGE frame clauses. Windowed `stddev_samp()` works too, so Bollinger can
use it directly instead of the `sqrt(avg(x*x) - avg(x)^2)` formula below
(that formula is still required inside live views).

Time filters below use TICK strings stored in `DECLARE` variables
(`@lookback := '$now - 1M..$now'`). In a Web Console notebook, replace them
with the notebook's `@timeFilter` or its own variables. For a live,
per-row-maintained version of SMA, Bollinger or EMA, see live views in
`views-and-storage.md`.

---

## Grafana-Ready Indicator Queries (trades table)

Copy-paste these into Grafana panels. They build OHLC from the raw `trades` table
using SAMPLE BY, so no pre-built candle table is needed.

### ATR: Grafana-ready

```sql
WITH ohlc AS (
  SELECT ts, symbol,
    first(price) AS open, max(price) AS high,
    min(price) AS low, last(price) AS close
  FROM trades
  WHERE $__timeFilter(ts) AND symbol = '$symbol'
  SAMPLE BY $__interval
),
with_prev AS (
  SELECT ts, symbol, high, low, close,
    lag(close) OVER (ORDER BY ts) AS prev_close
  FROM ohlc
),
true_range AS (
  SELECT ts, symbol, high, low, close,
    greatest(high - low, abs(high - prev_close), abs(low - prev_close)) AS tr
  FROM with_prev
  WHERE prev_close IS NOT NULL
)
SELECT ts AS time, tr AS true_range,
  AVG(tr) OVER (ORDER BY ts ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS atr
FROM true_range;
```

This is an SMA-smoothed ATR. For the classic Wilder ATR use
`avg(tr, 'period', 27) OVER (ORDER BY ts)`.

### OBV: Grafana-ready

```sql
WITH ohlc AS (
  SELECT ts, symbol,
    last(price) AS close, sum(amount) AS volume
  FROM trades
  WHERE $__timeFilter(ts) AND symbol = '$symbol'
  SAMPLE BY $__interval
),
with_direction AS (
  SELECT ts, symbol, close, volume,
    CASE
      WHEN close > lag(close) OVER (ORDER BY ts) THEN volume
      WHEN close < lag(close) OVER (ORDER BY ts) THEN -volume
      ELSE 0
    END AS directed_volume
  FROM ohlc
)
SELECT ts AS time,
  volume,
  sum(directed_volume) OVER (ORDER BY ts
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS obv
FROM with_direction;
```

### MACD: Grafana-ready

```sql
WITH ohlc AS (
  SELECT ts, symbol, last(price) AS close
  FROM trades
  WHERE $__timeFilter(ts) AND symbol = '$symbol'
  SAMPLE BY $__interval
),
ema AS (
  SELECT ts, close,
    avg(close, 'period', 12) OVER (ORDER BY ts) AS ema12,
    avg(close, 'period', 26) OVER (ORDER BY ts) AS ema26
  FROM ohlc
),
macd_line AS (
  SELECT ts, close, ema12, ema26,
    ema12 - ema26 AS macd
  FROM ema
),
with_signal AS (
  SELECT ts, close, macd,
    avg(macd, 'period', 9) OVER (ORDER BY ts) AS signal
  FROM macd_line
)
SELECT ts AS time,
  round(macd, 6) AS macd,
  round(signal, 6) AS signal,
  round(macd - signal, 6) AS histogram
FROM with_signal;
```


---

## Standalone Recipes (demo tables)

## MACD (Moving Average Convergence Divergence)

Trend-following momentum: EMA12 - EMA26 with 9-period signal line.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH ema AS (
  SELECT timestamp, symbol, close,
    avg(close, 'period', 12) OVER (PARTITION BY symbol ORDER BY timestamp) AS ema12,
    avg(close, 'period', 26) OVER (PARTITION BY symbol ORDER BY timestamp) AS ema26
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
macd_line AS (
  SELECT timestamp, symbol, close, ema12, ema26,
    ema12 - ema26 AS macd
  FROM ema
),
with_signal AS (
  SELECT timestamp, symbol, close, macd,
    avg(macd, 'period', 9) OVER (PARTITION BY symbol ORDER BY timestamp) AS signal
  FROM macd_line
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(macd, 6) AS macd,
  round(signal, 6) AS signal,
  round(macd - signal, 6) AS histogram
FROM with_signal
ORDER BY timestamp;
```

Classic params: 12/26/9. Faster: 8/17/9. Slower: 19/39/9.

---

## Stochastic Oscillator

Position of close within recent high-low range (0-100). %K > 80 = overbought, < 20 = oversold.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH ranges AS (
  SELECT timestamp, symbol, close,
    min(low) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS lowest_low,
    max(high) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS highest_high
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
with_k AS (
  SELECT timestamp, symbol, close,
    (close - lowest_low) / (highest_high - lowest_low) * 100 AS pct_k
  FROM ranges
  WHERE highest_high > lowest_low
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(pct_k, 2) AS pct_k,
  round(avg(pct_k) OVER (PARTITION BY symbol ORDER BY timestamp
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS pct_d
FROM with_k
ORDER BY timestamp;
```

This is Fast Stochastic. For Slow: smooth %K with 3-period SMA first, then apply 3-period SMA for %D.

---

## Rate of Change (ROC)

Percentage change vs N periods ago. Oscillates around zero.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH with_lag AS (
  SELECT timestamp, symbol, close,
    lag(close, 12) OVER (PARTITION BY symbol ORDER BY timestamp) AS close_12_ago
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round((close - close_12_ago) / close_12_ago * 100, 4) AS roc
FROM with_lag
WHERE close_12_ago IS NOT NULL
ORDER BY timestamp;
```

Common periods: 9/12 (short-term), 25 (medium), 200 (long-term).

---

## ATR (Average True Range)

Volatility measure using true range (accounts for gaps). Used for stop-losses and position sizing.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH with_prev AS (
  SELECT timestamp, symbol, high, low, close,
    lag(close) OVER (PARTITION BY symbol ORDER BY timestamp) AS prev_close
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
true_range AS (
  SELECT timestamp, symbol, high, low, close,
    greatest(high - low, abs(high - prev_close), abs(low - prev_close)) AS tr
  FROM with_prev
  WHERE prev_close IS NOT NULL
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(tr, 6) AS true_range,
  round(avg(tr, 'period', 14) OVER (PARTITION BY symbol ORDER BY timestamp), 6) AS atr
FROM true_range
ORDER BY timestamp;
```

Stop-loss: `entry_price - 2 * atr`. Position sizing: `(account_size * 0.01) / atr`.

---

## Donchian Channels

Highest high / lowest low over N periods. Breakout system (Turtle Trading: 20-day entry, 10-day exit).

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH channels AS (
  SELECT timestamp, symbol, close,
    max(high) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS upper_channel,
    min(low) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS lower_channel
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(upper_channel, 5) AS upper_channel,
  round(lower_channel, 5) AS lower_channel,
  round((upper_channel + lower_channel) / 2, 5) AS middle_channel
FROM channels
ORDER BY timestamp;
```

---

## Keltner Channels

EMA ± ATR bands. Smoother than Bollinger (uses ATR instead of stddev). When Bollinger moves inside Keltner = "squeeze".

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH with_prev AS (
  SELECT timestamp, symbol, high, low, close,
    lag(close) OVER (PARTITION BY symbol ORDER BY timestamp) AS prev_close
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
with_tr AS (
  SELECT timestamp, symbol, high, low, close,
    greatest(high - low, abs(high - prev_close), abs(low - prev_close)) AS tr
  FROM with_prev
  WHERE prev_close IS NOT NULL
),
with_indicators AS (
  SELECT timestamp, symbol, close,
    avg(close, 'period', 20) OVER (PARTITION BY symbol ORDER BY timestamp) AS ema20,
    avg(tr, 'period', 20) OVER (PARTITION BY symbol ORDER BY timestamp) AS atr
  FROM with_tr
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(ema20, 5) AS middle,
  round(ema20 + 2 * atr, 5) AS upper,
  round(ema20 - 2 * atr, 5) AS lower
FROM with_indicators
ORDER BY timestamp;
```

---

## Bollinger BandWidth

Measures band width as percentage. Low values = squeeze (breakout likely). Uses 6M history for range context.

```sql
DECLARE
  @symbol := 'EURUSD',
  @history := '$now - 6M..$now',
  @display := '$now - 1M..$now'

WITH bands AS (
  SELECT timestamp, symbol, close,
    AVG(close) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma20,
    AVG(close * close) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS avg_close_sq
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @history
),
bollinger AS (
  SELECT timestamp, symbol, close, sma20,
    sma20 + 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS upper_band,
    sma20 - 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS lower_band
  FROM bands
),
with_bandwidth AS (
  SELECT timestamp, symbol, close, sma20, upper_band, lower_band,
    (upper_band - lower_band) / sma20 * 100 AS bandwidth
  FROM bollinger
),
with_range AS (
  SELECT *, min(bandwidth) OVER (PARTITION BY symbol) AS min_bw,
    max(bandwidth) OVER (PARTITION BY symbol) AS max_bw
  FROM with_bandwidth
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(bandwidth, 4) AS bandwidth,
  round((bandwidth - min_bw) / (max_bw - min_bw) * 100, 1) AS range_position
FROM with_range
WHERE timestamp IN @display
ORDER BY timestamp;
```

range_position: 0% = at 6M minimum (squeeze), 100% = at 6M maximum.

---

## Realized Volatility (Annualized)

Rolling stddev of log returns, annualized. Compare with implied vol from options.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH returns AS (
  SELECT timestamp, symbol, close,
    ln(close / lag(close) OVER (PARTITION BY symbol ORDER BY timestamp)) AS log_return
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
with_stats AS (
  SELECT timestamp, symbol, close, log_return,
    avg(log_return) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS mean_return,
    avg(log_return * log_return) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS mean_sq_return
  FROM returns
  WHERE log_return IS NOT NULL
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(log_return * 100, 4) AS return_pct,
  round(sqrt(mean_sq_return - mean_return * mean_return) * sqrt(252 * 96) * 100, 2) AS realized_vol_annualized
FROM with_stats
ORDER BY timestamp;
```

Annualization factor: `sqrt(trading_days * periods_per_day)`. Real FX (24/5): `252 * 96` for 15m bars.
Daily bars: `sqrt(252)`. 24/7 data: `365 * 96`.

---

## OBV (On-Balance Volume)

Cumulative volume weighted by price direction. Divergence from price signals weakening trend.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH with_direction AS (
  SELECT timestamp, symbol, close,
    total_volume AS volume,
    CASE
      WHEN close > lag(close) OVER (PARTITION BY symbol ORDER BY timestamp) THEN volume
      WHEN close < lag(close) OVER (PARTITION BY symbol ORDER BY timestamp) THEN -volume
      ELSE 0
    END AS directed_volume
  FROM fx_trades_ohlc_1m
  WHERE symbol = @symbol AND timestamp IN @lookback
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(volume, 0) AS volume,
  round(sum(directed_volume) OVER (PARTITION BY symbol ORDER BY timestamp
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 0) AS obv
FROM with_direction
ORDER BY timestamp;
```

Absolute OBV value is meaningless: only direction and divergence matter.

---

## Volume Profile

Distribution of volume across price levels. Uses `floor()` for binning.

```sql
-- Fixed tick size
DECLARE @tick_size := 0.01
SELECT
  floor(price / @tick_size) * @tick_size AS price_bin,
  round(SUM(quantity), 2) AS volume
FROM fx_trades
WHERE symbol = 'EURUSD' AND timestamp IN '$today'
ORDER BY price_bin;
```

```sql
-- Dynamic 50-bin distribution
WITH raw_data AS (
  SELECT price, quantity
  FROM fx_trades
  WHERE symbol = 'EURUSD' AND timestamp IN '$today'
),
tick_size AS (
  SELECT (max(price) - min(price)) / 49 as tick_size
  FROM raw_data
)
SELECT
  floor(price / tick_size) * tick_size AS price_bin,
  round(SUM(quantity), 2) AS volume
FROM raw_data CROSS JOIN tick_size
ORDER BY 1;
```

QuestDB does implicit GROUP BY on non-aggregated columns.

---

## Volume Spike Detection

Flag candles where volume > 2x previous candle.

```sql
DECLARE
  @range := '$now - 7h..$now',
  @symbol := 'EURUSD'

WITH candles AS (
  SELECT timestamp, symbol, sum(quantity) AS volume
  FROM fx_trades
  WHERE timestamp IN @range AND symbol = @symbol
  SAMPLE BY 30s
),
prev_volumes AS (
  SELECT timestamp, symbol, volume,
    LAG(volume) OVER (PARTITION BY symbol ORDER BY timestamp) AS prev_volume
  FROM candles
)
SELECT *,
  CASE WHEN volume > 2 * prev_volume THEN 'spike' ELSE 'normal' END AS spike_flag
FROM prev_volumes;
```

---

## Maximum Drawdown

Largest peak-to-trough decline. Key risk metric.

```sql
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1M..$now'

WITH with_peak AS (
  SELECT timestamp, symbol, close,
    max(close) OVER (PARTITION BY symbol ORDER BY timestamp
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_peak
  FROM market_data_ohlc_15m
  WHERE symbol = @symbol AND timestamp IN @lookback
),
with_drawdown AS (
  SELECT timestamp, symbol, close, running_peak,
    (close - running_peak) / running_peak * 100 AS drawdown
  FROM with_peak
)
SELECT timestamp, symbol,
  round(close, 5) AS close,
  round(running_peak, 5) AS peak,
  round(drawdown, 4) AS drawdown_pct,
  round(min(drawdown) OVER (PARTITION BY symbol ORDER BY timestamp
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 4) AS max_drawdown_pct
FROM with_drawdown
ORDER BY timestamp;
```

---

## Bid-Ask Spread

Liquidity measure. Narrow = liquid. Wide = illiquid or stressed.

```sql
-- Tick-level spread
DECLARE
  @symbol := 'EURUSD',
  @lookback := '$now - 1h..$now'

SELECT timestamp, symbol,
  round(bid_price, 5) AS bid,
  round(ask_price, 5) AS ask,
  round(ask_price - bid_price, 6) AS spread_absolute,
  round((ask_price - bid_price) / ((bid_price + ask_price) / 2) * 10000, 2) AS spread_bps,
  round((bid_price + ask_price) / 2, 5) AS mid_price
FROM core_price
WHERE symbol = @symbol AND timestamp IN @lookback
ORDER BY timestamp;
```

```sql
-- Hourly aggregated spread
SELECT timestamp, symbol,
  round(avg((ask_price - bid_price) / ((bid_price + ask_price) / 2) * 10000), 2) AS avg_spread_bps,
  round(max((ask_price - bid_price) / ((bid_price + ask_price) / 2) * 10000), 2) AS max_spread_bps,
  count() AS quote_count
FROM core_price
WHERE symbol = @symbol AND timestamp IN '$now - 1d..$now'
SAMPLE BY 1h
ORDER BY timestamp;
```

FX majors: typically 0.1-1.0 bps. FX minors: 1-5 bps. Crypto: 1-50+ bps.

---

## Liquidity Comparison (L2Price)

Effective spread at a given order size using Level 2 order book data.

```sql
-- Compare effective spread across instruments at 100K size
WITH latest_books AS (
  SELECT timestamp, symbol, bids, asks
  FROM market_data
  WHERE timestamp IN '$today'
  LATEST ON timestamp PARTITION BY symbol
)
SELECT symbol,
  L2PRICE(100_000, asks[2], asks[1]) AS buy_price,
  L2PRICE(100_000, bids[2], bids[1]) AS sell_price,
  L2PRICE(100_000, asks[2], asks[1]) - L2PRICE(100_000, bids[2], bids[1]) AS effective_spread,
  (L2PRICE(100_000, asks[2], asks[1]) - L2PRICE(100_000, bids[2], bids[1])) /
    ((L2PRICE(100_000, asks[2], asks[1]) + L2PRICE(100_000, bids[2], bids[1])) / 2) * 10_000 AS spread_bps
FROM latest_books
ORDER BY spread_bps;
```

`L2PRICE(size, price_array, qty_array)` calculates average execution price across multiple levels.

---

## Aggressor Volume Imbalance

Buy vs sell aggressor volume ratio. Requires `side` column on trades.

```sql
WITH volumes AS (
  SELECT symbol,
    SUM(CASE WHEN side = 'buy' THEN amount ELSE 0 END) AS buy_volume,
    SUM(CASE WHEN side = 'sell' THEN amount ELSE 0 END) AS sell_volume
  FROM trades
  WHERE timestamp IN '$yesterday'
    AND symbol IN ('ETH-USDT', 'BTC-USDT')
)
SELECT symbol, buy_volume, sell_volume,
  ((buy_volume - sell_volume)::double / (buy_volume + sell_volume)) * 100 AS imbalance
FROM volumes;
```

Imbalance: -100% (all sell) to +100% (all buy). 0% = balanced.
