# Grafana with QuestDB

Use this only when the user asks for Grafana. The default for QuestDB
dashboards is a Web Console notebook built through the MCP server
(`dashboards-mcp.md`).

QuestDB has a dedicated Grafana plugin, `questdb-questdb-datasource`, which
connects over PGWire (8812). Docs:
`https://questdb.com/docs/integrations/visualization/grafana.md`,
recipes under `https://questdb.com/docs/cookbook/integrations/grafana/`.

## Containers and Datasource

Credentials come from the environment (`GRAFANA_USER`, `GRAFANA_PASSWORD`);
never write them into commands or files.

```bash
docker run -d --name grafana -p 3000:3000 -e GF_INSTALL_PLUGINS=questdb-questdb-datasource -e GF_SECURITY_ADMIN_PASSWORD=$GRAFANA_PASSWORD -e GF_DASHBOARDS_MIN_REFRESH_INTERVAL=250ms grafana/grafana:latest
```
```bash
curl -sf --retry 30 --retry-delay 1 --retry-all-errors -o /dev/null http://localhost:3000/api/health
```
```bash
curl -s -X POST http://localhost:3000/api/datasources -u "$GRAFANA_USER:$GRAFANA_PASSWORD" -H "Content-Type: application/json" -d '{"name":"QuestDB","type":"questdb-questdb-datasource","access":"proxy","jsonData":{"server":"host.docker.internal","port":8812,"username":"admin","tlsMode":"disable","timeout":"120","queryTimeout":"60"},"secureJsonData":{"password":"quest"}}'
```

The plugin uses `jsonData`, not the standard `url` field:
- `server`: hostname only, no port, no protocol (`host.docker.internal` from a container)
- `port`: integer, separate (`8812`)
- `tlsMode`: `"disable"` for local Docker; omitting it enables TLS and the connection fails
- `username` + `secureJsonData.password`: `admin`/`quest` by default
- Reference the datasource by `{"uid": ..., "type": "questdb-questdb-datasource"}`, never by name

Grafana blocks refresh intervals under 5s unless started with
`GF_DASHBOARDS_MIN_REFRESH_INTERVAL=250ms`. Open dashboard URLs with
`?refresh=250ms&from=now-5m&to=now`; otherwise the JSON refresh is ignored.

## Query Rules

- Time filter: `WHERE $__timeFilter(ts)`. Bucket: `SAMPLE BY $__interval` or
  a fixed bar size (`5s` default for real-time crypto panels).
- Grafana needs a `time` column: alias `ts AS time` in the **final** SELECT
  only. Put every `OVER()` in a CTE where `ts` is still `ts`.
- Panel target `"format": 1` (integer). The string `"table"` from Grafana's
  JSON export fails with `cannot unmarshal string`.
- Variables: `'$symbol'` or `'${SYMBOL}'`. Dropdown query:
  `SELECT DISTINCT symbol FROM trades ORDER BY symbol`. For tables with many
  stale symbols:
  `SELECT DISTINCT symbol FROM (SELECT symbol FROM trades WHERE timestamp IN '$now-4d..$now' LATEST ON timestamp PARTITION BY symbol) ORDER BY symbol`.
- Grafana's time range can feed `DECLARE`: see
  `https://questdb.com/docs/cookbook/integrations/grafana/declare-time-range.md`.

## Panel Patterns

**Indicators on the candlestick panel.** One `candlestick` panel
(`mode: "candles+volume"`, `includeAllFields: true`). refId A returns
`time, open, high, low, close, volume`; refIds B, C, D each return
`time` plus their indicator columns only (VWAP, Bollinger, RSI). Returning
OHLC from B/C/D breaks the candles.

**Overrides are not cosmetic.** Without them RSI (0-100) and price share an
axis and the candles flatten:

```json
"overrides": [
  {"matcher": {"id": "byName", "options": "volume"},
   "properties": [{"id": "custom.axisPlacement", "value": "hidden"}]},
  {"matcher": {"id": "byName", "options": "vwap"},
   "properties": [{"id": "color", "value": {"fixedColor": "orange", "mode": "fixed"}}]},
  {"matcher": {"id": "byName", "options": "upper_band"},
   "properties": [{"id": "custom.lineStyle", "value": {"fill": "dash", "dash": [10, 10]}},
                  {"id": "custom.fillBelowTo", "value": "lower_band"},
                  {"id": "custom.fillOpacity", "value": 8}]},
  {"matcher": {"id": "byFrameRefID", "options": "D"},
   "properties": [{"id": "custom.axisPlacement", "value": "right"},
                  {"id": "min", "value": 0}, {"id": "max", "value": 100},
                  {"id": "unit", "value": "percent"}]}
]
```

Spread next to bid/ask needs `custom.axisPlacement: "right"` on `spread`.

**Per-panel time ranges:** `"timeFrom": "1m", "hideTimeOverride": true`
(`"30m"`, `"now/d"`); `"timeShift": "5s"` to hide the still-filling bucket.

**Repeat per symbol:** `"repeat": "SYMBOL", "repeatDirection": "h", "maxPerRow": 2`.

**Depth chart:** the `ae3e-plotly-panel` plugin with the latest book snapshot:

```sql
WITH snapshot AS (
  SELECT timestamp, bids, asks FROM market_data
  WHERE $__timeFilter(timestamp) AND symbol = '${SYMBOL}'
  LATEST ON timestamp PARTITION BY symbol
)
SELECT timestamp,
       bids[1] AS bprices, array_cum_sum(bids[2]) AS bcumvolumes,
       asks[1] AS aprices, array_cum_sum(asks[2]) AS acumvolumes
FROM snapshot;
```

## Ready Queries (crypto pipeline schema from `crypto-pipeline.md`)

```sql
-- A: OHLC
SELECT ts AS time, first(price) AS open, max(price) AS high, min(price) AS low,
       last(price) AS close, sum(amount) AS volume
FROM trades WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY 5s;

-- B: VWAP (cumulative over the visible range)
WITH ohlc AS (
  SELECT ts, max(price) AS high, min(price) AS low, last(price) AS close, sum(amount) AS volume
  FROM trades WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY 5s
), v AS (
  SELECT ts, sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE)
           / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap
  FROM ohlc
)
SELECT ts AS time, vwap FROM v;

-- C: Bollinger (20, 2)
WITH ohlc AS (
  SELECT ts, last(price) AS close FROM trades
  WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY 5s
), s AS (
  SELECT ts,
         avg(close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma20,
         stddev_samp(close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sd20
  FROM ohlc
)
SELECT ts AS time, sma20, sma20 + 2 * sd20 AS upper_band, sma20 - 2 * sd20 AS lower_band FROM s;

-- D: RSI (14)
WITH ohlc AS (
  SELECT ts, last(price) AS close FROM trades
  WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY 5s
), ch AS (
  SELECT ts, close - lag(close) OVER (ORDER BY ts) AS change FROM ohlc
), gl AS (
  SELECT ts, CASE WHEN change > 0 THEN change ELSE 0 END AS gain,
             CASE WHEN change < 0 THEN -change ELSE 0 END AS loss FROM ch
), a AS (
  SELECT ts, avg(gain) OVER (ORDER BY ts ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_gain,
             avg(loss) OVER (ORDER BY ts ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_loss
  FROM gl
)
SELECT ts AS time,
       CASE WHEN avg_loss = 0 THEN 100 ELSE 100 - 100 / (1 + avg_gain / avg_loss) END AS rsi
FROM a;

-- Spread panel (2D order book arrays)
SELECT ts AS time, avg(asks[1,1] - bids[1,1]) AS spread,
       avg(bids[1,1]) AS best_bid, avg(asks[1,1]) AS best_ask
FROM orderbook WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY 5s;
```

## Deploying a Dashboard via the API

```python
import os, requests, subprocess, sys

GRAFANA_URL = "http://localhost:3000"
AUTH = (os.environ["GRAFANA_USER"], os.environ["GRAFANA_PASSWORD"])
ds = [d for d in requests.get(f"{GRAFANA_URL}/api/datasources", auth=AUTH).json()
      if d["type"] == "questdb-questdb-datasource"]
DS = {"uid": ds[0]["uid"], "type": "questdb-questdb-datasource"}

def target(ref, sql):
    return {"refId": ref, "datasource": DS, "format": 1, "rawSql": sql}

dashboard = {"dashboard": {
    "title": "Crypto Real-Time", "uid": "crypto-realtime", "timezone": "browser",
    "refresh": "250ms", "liveNow": False, "schemaVersion": 38,
    "time": {"from": "now-5m", "to": "now"},
    "timepicker": {"refresh_intervals": ["250ms", "500ms", "1s", "5s", "10s", "30s", "1m"]},
    "templating": {"list": [{"name": "symbol", "type": "query", "datasource": DS,
        "query": "SELECT DISTINCT symbol FROM trades ORDER BY symbol;", "refresh": 1,
        "current": {"text": "BTC-USDT", "value": "BTC-USDT"}}]},
    "panels": [
        {"id": 1, "type": "candlestick", "title": "OHLC + Indicators - $symbol",
         "gridPos": {"h": 12, "w": 24, "x": 0, "y": 0}, "datasource": DS,
         "options": {"mode": "candles+volume", "includeAllFields": True,
                     "candleStyle": "candles", "colorStrategy": "open-close",
                     "fields": {"open": "open", "high": "high", "low": "low",
                                "close": "close", "volume": "volume"}},
         "fieldConfig": {"defaults": {}, "overrides": OVERRIDES},   # the overrides above
         "targets": [target("A", OHLC_SQL), target("B", VWAP_SQL),
                     target("C", BOLLINGER_SQL), target("D", RSI_SQL)]},
        {"id": 2, "type": "timeseries", "title": "Spread - $symbol",
         "gridPos": {"h": 6, "w": 24, "x": 0, "y": 12}, "datasource": DS,
         "fieldConfig": {"defaults": {}, "overrides": [
             {"matcher": {"id": "byName", "options": "spread"},
              "properties": [{"id": "custom.axisPlacement", "value": "right"}]}]},
         "targets": [target("A", SPREAD_SQL)]},
    ]}, "overwrite": True}

resp = requests.post(f"{GRAFANA_URL}/api/dashboards/db", auth=AUTH, json=dashboard)
url = f"{GRAFANA_URL}{resp.json()['url']}?refresh=250ms&from=now-5m&to=now"
subprocess.run(["open" if sys.platform == "darwin" else "xdg-open", url])
```

Put each SQL string on one line in the Python file. Wait for data before
deploying (poll `SELECT count() FROM trades` until it is non-zero) so the
panels do not open empty.
