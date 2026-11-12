# Dashboards in the QuestDB Web Console (MCP)

The preferred way to build QuestDB dashboards is a **Web Console notebook**,
authored by the agent through the QuestDB MCP server. Notebooks combine SQL,
markdown and charts, refresh live per cell, take variables and a time range,
and need no extra service (no Grafana, no datasource, no plugin). Use Grafana
only when the user asks for it (see `grafana.md`).

Docs: `https://questdb.com/docs/getting-started/web-console/mcp-server.md`,
`.../web-console/notebooks/overview.md` (also `cells`, `charts`, `variables`,
`live-dashboards`, `manage-share`).

---

## Setup and Pairing

Install once (Node.js required). The wizard detects Claude Code, Codex,
Cursor, OpenCode and Gemini CLI and writes their MCP config:

```bash
npx @questdb/mcp-server-questdb@latest setup
```

The console shows the exact pinned version to use in its MCP status pill.
The old package name `@questdb/mcp-bridge` is obsolete. Manual config
(`.mcp.json` in the project root):

```json
{"mcpServers":{"questdb":{"command":"npx","args":["-y","@questdb/mcp-server-questdb@latest"]}}}
```

The Web Console must be open in a browser (default `http://127.0.0.1:9000`,
override with the `CONSOLE_ORIGIN` env var). Pairing, every session:

1. Call `get_pairing_credentials`.
2. Show its `userMessage` to the user **verbatim** (deep link plus ws_url and
   token). Never skip this.
3. In the same turn call `wait_for_pairing`; it blocks up to 50 s. Retry on
   `timeout` up to about 10 times.

Unpaired tools return `BRIDGE_NOT_PAIRED`. The user picks a permission level
in the console:

| Level | Schema tools | `run_query` returns rows | DDL/DML |
|-------|--------------|--------------------------|---------|
| None | no | no | no |
| Schema access | yes | no | no |
| Read | yes | yes | no |
| Write | yes | yes | yes (only via `run_cell`, after user consent) |

Notebook authoring works at every level. Drawing or running DQL cells never
needs `read`: the user sees the data, the agent does not. If a tool returns
`PERMISSION_DENIED`, tell the user which scope to grant.

---

## Tools You Will Use

Discover data: `get_tables`, `get_table_schema`, `get_table_details`,
`validate_query` (validate every query before putting it in a cell),
`get_questdb_toc` / `get_questdb_documentation`, `run_query` (returns rows,
needs `read`).

Workspace and notebooks: `get_workspace_state` (call first in every notebook
turn; a freshness gate applies), `create_notebook(label)` (always opens in the
background), `activate_notebook` / `activate_tab` (only after the user agrees),
`duplicate_notebook`, `get_notebook_state`, `list_cells`, `get_cell`.

Tabs (newer consoles): `rename_tab`, `archive_tab` (close, reversible),
`restore_tab`, `delete_tab` (permanent, only when the user asks). Older
consoles have `delete_notebook` instead. Use whichever your tool list shows.

Cells: `add_cell`, `update_cell`, `delete_cell`, `move_cell_up/down`,
`duplicate_cell`, `run_cell`, `set_cell_name` (also the chart title),
`set_cell_mode` (`run` = table, `draw` = chart), `set_cell_chart_config`,
`set_cell_highlight_config`, `set_cell_autorefresh`,
`set_notebook_autorefresh`, `set_layout_mode`, `set_cell_layout`,
`set_cell_dimensions`, `set_cell_maximized`.

**Build a whole dashboard with one `apply_notebook_state` call.** It is a full
PUT of the notebook: cells, names, modes, chart configs, highlight rules, grid
positions, per-cell refresh, variables and the refresh default.

### Rules That Bite

- `apply_notebook_state.cells` is the COMPLETE list. Existing cells you omit
  are deleted. Omitting `chart_config` removes a chart; `name: null` clears a
  name. Use `preserve_value: true` for cells whose SQL you are not changing.
- Never write back a truncated value (`get_cell` caps at 4 KB unless
  `get_full_content: true`; previews are 120 chars).
- `chart_config.queries` must have exactly one entry per `;`-separated
  statement in the cell, or the call is rejected.
- Candlesticks need an explicit `ohlc: {open, high, low, close}` mapping.
- Draw cells auto-execute. Do not call `run_cell` after setting draw mode.
  In a background tab they render when the user opens it: `runs[]` lists only
  run-mode cells, and draw cells report `last_run_status: "none"` until then.
  Validate chart SQL with `run_query` (one statement per call; a cell's
  `;`-separated statements one at a time) before applying.
- DDL/DML is never auto-run or auto-refreshed. Create tables and views with
  `run_query` (write scope) or a cell the user runs, not in a dashboard cell.
- **Nothing polls until a refresh value is set.** Set
  `auto_refresh_default` (or `set_notebook_autorefresh`) on every live
  dashboard.
- Grid layout is 12 columns: `grid: {x, y, w}` with `w` 1-12. Height comes
  from `editor_height` / `result_height` (charts need `result_height` >= 296,
  tables >= 100). Use `view: "result"` to hide editors on a dashboard; never
  send `view: "editor"` to an existing cell, it discards its result.
- Markdown cells: `type: "markdown"`, the value is markdown, `mode` and
  `chart_config` must be null. `editor_height: 86` gives a clean title cell.
- Limits: 200 cells per notebook, 100 tabs, 10,000 rows per statement.

---

## Charts

A draw-mode cell can hold several `;`-separated SELECTs. They **combine into
one chart** sharing the first query's x-axis (all temporal: merged by time;
all categorical: merged by category). Each query has its own config:

```json
{"type": "candlestick", "y_columns": null,
 "ohlc": {"open": "open", "high": "high", "low": "low", "close": "close"},
 "partition_by_column": null, "axis": null, "enabled": null, "name": "OHLC 15m"}
```

- `type`: `line`, `area`, `stepLine`, `stepArea`, `bar`, `stackedBar`,
  `scatter`, `pie`, `candlestick`. `pie`, `scatter` and `stackedBar` render
  only as single-query charts.
- `y_columns`: columns to plot (null = all numeric). Not used for candlestick.
- `partition_by_column`: one series per value of a categorical column
  (e.g. one line per table or symbol) without pivoting in SQL.
- `axis: "right"` puts the query on the shared secondary axis;
  `right_axis: {name, min, max}` pins its title and range. Newer consoles
  also have `left_axis: {name, min, max}`.
- A **numeric** `x_column` (not a timestamp) gives a continuous value axis,
  used for depth charts and histograms.
- Newer consoles accept a `volume` column on a candlestick query, drawn as a
  sub-pane under the candles. Use it when the schema offers it; otherwise use
  the right-axis trick below.

### Highlight Rules (run-mode grids)

`set_cell_highlight_config` colors live tables:
- `kind: "previous"` + `identity_columns: ["symbol"]` flashes cells that moved
  up (`op: "gt"`, `color: "green"`) or down (`op: "lt"`, `color: "red"`).
- `kind: "value"`, `op: "between"`, `fill: "gradient"`, `value`/`to` null:
  shade a column across its current min..max.
  The gradient is linear, so on heavy-tailed columns (trade notional, order
  size) one outlier takes the far color and every other cell looks the same.
- `kind: "steps"`: thresholds, e.g. book level 1 green, 2-3 amber, 4+ red.
  Prefer steps for skewed values (notional `1000` / `10000` / `100000`).
- `kind: "newRow"`: paint rows that just appeared.
- Colors: red, teal, amber, lime, orange, purple, green, pink, blue, olive.

---

## Variables and Time Range

Notebook variables are sent to the server as a `DECLARE` block prepended to
each SELECT (merged with a cell's own `DECLARE`; the cell's names win).
Reference them as `@name`. They are not injected into INSERT/CREATE/ALTER.

```json
"variables": [
  {"name": "symbol", "value": "'EURUSD'"},
  {"name": "asof",   "value": "now()"}
]
```

- Names have no `@`. Values are raw SQL expressions: quote strings yourself
  (`"'EURUSD'"`), `now()`, `dateadd('d', -7, now())`, `'$now-1h..$now'`.
- Order matters: a variable may reference earlier ones.
- A value list works with `IN`: `{"name": "syms", "value": "('EURUSD','GBPUSD')"}`
  then `WHERE symbol IN @syms`.
- Never name a CTE or alias after a SQL keyword (`lt`, `asof`, `sample`,
  `splice`, `horizon`). The console prepends the `DECLARE` only when its own
  parser accepts the cell, and these names make that parser fail (the server
  itself accepts them). The cell then fails with a misleading
  `Invalid column: @name`, or, if it declares its own default for the
  variable, silently runs with that default. Use names like `fills_lt`, or
  quote them (`"lt"`).

**Newer consoles** (check the `apply_notebook_state` schema for `kind`,
`time_range` and `set_cell_time_range`) add:
- Typed variables: `kind: "expression"` (fixed SQL), `"text"` (textbox the
  user edits, inserted as written), `"list"` (picker fed by a query or a typed
  list; multi-select renders `(v1, v2)` for `IN @name`; optional "All").
- **Global variables** shared by every notebook: `get_global_variables`,
  `apply_global_variables(variables, expected_revision, time_range)`. Names
  are unique across globals and notebook variables, ignoring case. The call
  replaces the whole ordered list, so read first and pass the revision.
- A **notebook time range** (`time_range: {from, to}` with Grafana-style
  bounds such as `now-15m`, `now/d`, `now-1d/d`, or ISO timestamps) that
  declares `@timeFrom`, `@timeTo` and `@timeFilter` for every cell:
  `WHERE ts IN @timeFilter`. These three names are reserved.
- Per-cell `time_range` override and `time_shift` (e.g. `-1d` to show
  yesterday next to today), via `set_cell_time_range` or bulk apply.

When the console has no time-range support, drive ranges with plain variables
(`@asof`, `@lookback`) as in the FX example below.

---

## Refresh Per Cell

Give each cell the cadence its data deserves, and set a notebook default:

| Cell | Refresh |
|------|---------|
| Order book depth, top of book | `200ms`-`250ms` |
| 1-minute candles on 1s bars, spread table | `500ms` |
| Hourly indicators, intraday VWAP/RSI | `30s` |
| Instance metrics (`sys.metrics`) | `15s`-`1m` |

Values: `true` (adaptive), `false` (off), or `"<n>ms|s|m"` from 50ms to 60m.
Charts poll only while visible and the tab is in the foreground.

---

## Reference Dashboards (complete, working exports)

Two full notebook exports ship with this skill. Read them when building a
similar dashboard, and copy their SQL, chart configs, refresh intervals and
layout:

- `examples/fx-live-dashboard.json`: live FX order book dashboard for the
  `demo.questdb.io` dataset (depth chart, OHLC + volume overlay, indicators,
  highlighted live tables).
- `examples/monitoring-dashboard.json`: instance health from `sys.metrics`
  and `sys.telemetry_wal` (25 cells).

Structure: `data.data[0].rows[0]` is the tab (`label`), and
`notebookViewState` holds `cells[]` (`value` = SQL or markdown, `name`,
`type`, `mode`, `chartConfig`, `highlightConfig`, `autoRefresh`) and
`settings` (`layout[]` grid positions keyed by cell id, `variables[]`,
`autoRefreshDefault`). The export uses camelCase (`xColumn`, `yColumns`,
`rightAxis`, `partitionByColumn`); the MCP tools take the same fields in
snake_case (`x_column`, `y_columns`, `right_axis`, `partition_by_column`).
The user can also import either file directly: tab bar menu > **Import tabs**.

## Pattern: Live FX Dashboard (demo dataset)

Works against `demo.questdb.io` tables `market_data` (2D `bids`/`asks`
arrays), `core_price`, `bbo_1s`, `bbo_1h`, `market_data_ohlc_1m`,
`market_data_ohlc_15m`. Variables: `symbol := 'EURUSD'`, `asof := now()`.
Setting `asof` to a past timestamp replays the whole dashboard at that moment,
which is why the filters are written relative to `@asof`
(`timestamp >= dateadd('m', -1, @asof) AND timestamp <= @asof`) instead of
`$now`.

Grid layout (w=6 halves), notebook default `250ms`.

**1. Market depth, constant width (stepArea, numeric x, 200ms).** Cumulative
depth for 7 levels per side, x in pips from mid. Two extra points at
`±edge` (10 level spacings) carrying the level-7 depth keep the x-range
fixed, so the chart does not jump in width on every refresh.

```sql
WITH book AS (
  SELECT bids, asks FROM market_data
  WHERE symbol = @symbol AND timestamp >= dateadd('m', -10, @asof) AND timestamp <= @asof
  LATEST ON timestamp PARTITION BY symbol
), m0 AS (
  SELECT (bids[1][1] + asks[1][1]) / 2 AS mid,
         CASE WHEN @symbol LIKE '%JPY' THEN 0.01 WHEN @symbol = 'XAGUSD' THEN 0.001
              WHEN @symbol LIKE 'X%' THEN 0.01 ELSE 0.0001 END AS pip,
         bids[1][1] - bids[1][2] AS spacing,
         array_cum_sum(bids[2])[7] AS bid7, array_cum_sum(asks[2])[7] AS ask7
  FROM book
), m AS (
  SELECT mid, pip, round(10 * spacing / pip) AS edge, bid7, ask7 FROM m0
)
SELECT round((u.price - m.mid) / m.pip, 1) AS pips_from_mid, u.depth AS bid_depth, NULL::double AS ask_depth
FROM book b CROSS JOIN m, UNNEST(b.bids[1], array_cum_sum(b.bids[2])) WITH ORDINALITY u(price, depth, lvl)
WHERE u.lvl <= 7
UNION ALL
SELECT round((u.price - m.mid) / m.pip, 1), NULL::double, u.depth
FROM book b CROSS JOIN m, UNNEST(b.asks[1], array_cum_sum(b.asks[2])) WITH ORDINALITY u(price, depth, lvl)
WHERE u.lvl <= 7
UNION ALL SELECT -edge, bid7, NULL::double FROM m
UNION ALL SELECT edge, NULL::double, ask7 FROM m
ORDER BY pips_from_mid
```
`chart_config`: `x_column: "pips_from_mid"`, one query
`{type: "stepArea", y_columns: ["bid_depth", "ask_depth"]}`.

**2. OHLC with a volume overlay that does not swamp the candles (500ms).**
Two statements: candles on the left axis, volume bars on the right axis.
Volume is normalised to 0.1..1 and the right axis is pinned to `0..4`, so the
bars use the bottom quarter of the chart.

```sql
SELECT timestamp, first(bids[1,1]) AS open, max(bids[1,1]) AS high,
       min(bids[1,1]) AS low, last(bids[1,1]) AS close
FROM market_data
WHERE timestamp >= dateadd('m', -1, @asof) AND timestamp <= @asof AND symbol = @symbol
SAMPLE BY 1s;
SELECT timestamp,
  0.1 + 0.9 * (total_volume - min(total_volume) OVER ())
      / NULLIF(max(total_volume) OVER () - min(total_volume) OVER (), 0) AS volume
FROM (
  SELECT timestamp, sum(bids[2,1]) AS total_volume FROM market_data
  WHERE timestamp >= dateadd('m', -1, @asof) AND timestamp <= @asof AND symbol = @symbol
  SAMPLE BY 1s
)
```
`queries`: `[{type: "candlestick", ohlc: {...}}, {type: "bar", y_columns: ["volume"], axis: "right", name: "Volume"}]`,
`right_axis: {name: "Volume (relative)", min: 0, max: 4}`.
(On consoles with candlestick `volume` support, a single query returning a
`volume` column is simpler.)

**3. Indicators on one chart (30s).** Four statements: 15m candles,
cumulative VWAP from 1m candles (left axis), RSI (right axis pinned 0..100),
Bollinger bands (left axis). Intraday range:
`timestamp >= timestamp_floor('d', @asof) AND timestamp <= @asof`.
`right_axis: {name: "RSI %", min: 0, max: 100}`.

**4. Live tables with highlight rules (250-500ms).** Last 6 seconds of spread
and notional (`SAMPLE BY 1s ORDER BY ts DESC LIMIT 6`) with gradient fills;
top-of-book vs core price (`LIMIT -6` for the last rows, then
`ASOF JOIN market_data ON symbol` and `insertion_point(bids[2], bid_volume)`
to find the book level a quote's size reaches), with step colors on `Level`.

**5. BBO and mirrored volume (30s).** `bbo_1s` bid/ask/mid lines on the left;
`SAMPLE BY 30s` bid notional and **negated** ask notional on the right, so
buy and sell pressure sit above and below zero.

---

## Pattern: Instance Monitoring Dashboard

Built on `sys.metrics` and `sys.telemetry_wal` (see `monitoring.md` for
the columns and queries). Layout: a full-width markdown cell explaining how to
read the panels, then w=6 chart pairs. Default refresh `30s`, `15s` for the
most important, `1m` for slow gauges. Techniques:

- Counters become rates with `lag()` deltas, guarded against restarts:
  `sum(CASE WHEN d >= 0 THEN d ELSE 0 END) / nullif(sum(seconds), 0)`.
- One line per table with `partition_by_column: "table_name"` after
  `JOIN tables() t ON s.tableId = t.id`; exclude views with `NOT t.matView`.
- "Hot tables only": a CTE picks the top 5 by p95 over the last 5 minutes and
  joins back to the 2-hour series.
- Units in the cell name ("· p95 ms", "(GiB)", "/min"), conversions in SQL
  (`/1073741824.0`).

---

## Dashboard Checklist

1. `get_workspace_state`, then `get_tables` / `get_table_schema` for real column names.
2. `validate_query` every statement.
3. `create_notebook(label)`, then one `apply_notebook_state` with
   `layout_mode: "grid"`, `auto_refresh_default`, `variables`, and every cell
   (name, value, mode, chart_config, grid, view `"result"`, per-cell `auto_refresh`).
4. Check the `runs[]` in the response and fix any `ERROR:` statements
   (draw cells are not in `runs[]`; test their SQL with `run_query` first).
5. Tell the user the notebook is ready in a background tab and offer to open it
   (`activate_notebook` only if they say yes).
6. Mention they can export it (tab menu > Export) to share it as JSON.
