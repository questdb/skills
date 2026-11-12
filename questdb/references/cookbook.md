# QuestDB Cookbook Recipe Index

Official recipes, each fetchable as markdown. Paths are relative to
`https://questdb.com/docs/`:

```bash
curl -s "https://questdb.com/docs/cookbook/sql/finance/markout.md"
curl -s "https://questdb.com/docs/llms.txt"        # every page with its .md URL
```

Most recipes run against `demo.questdb.io` tables. Adapt table and column
names to the user's schema (see the mapping in `indicators.md`).

## Analysing a Dataset

When asked to find issues, anomalies or patterns without a specific question:

1. Profile first: row counts, time span, gaps, nulls (`count() - count(col)`),
   duplicates per key, and late or restated rows.
2. Break each metric down by every categorical column in the schema, and by
   pairs of them. An effect concentrated in one segment can vanish in the
   average of a mix.
3. Match time scales to the data. Derive horizons and bucket sizes from the
   gaps between events and how long orders or positions live, and compare
   several scales instead of stopping at a default.
4. Separate common moves from specific ones. Before blaming one entity
   (counterparty, venue, device, host), check whether the whole population
   moved at the same time, against a benchmark or its peers.
5. Report each finding with its size, the rows and period it affects, and the
   query that shows it.

## Finance: Post-Trade and Execution Quality (TCA)

All built on `HORIZON JOIN`; use them instead of hand-rolled ASOF self-joins.

| Recipe | Path |
|--------|------|
| Post-trade analysis overview (start here) | `cookbook/sql/finance/post-trade-overview.md` |
| Slippage per fill | `cookbook/sql/finance/slippage.md` |
| Slippage aggregated by venue and counterparty | `cookbook/sql/finance/slippage-aggregated.md` |
| Markout (reversion, adverse selection) | `cookbook/sql/finance/markout.md` |
| Last-look detection (ms markouts) | `cookbook/sql/finance/last-look.md` |
| Implementation shortfall decomposition | `cookbook/sql/finance/implementation-shortfall.md` |
| Order-level implementation shortfall | `cookbook/sql/finance/implementation-shortfall-order.md` |
| ECN scorecard (pivoted) | `cookbook/sql/finance/ecn-scorecard.md` |

## Finance: Indicators and Microstructure

| Recipe | Path |
|--------|------|
| OHLC bars | `cookbook/sql/finance/ohlc.md` |
| VWAP / TWAP | `cookbook/sql/finance/vwap.md`, `cookbook/sql/finance/twap.md` |
| Bollinger bands / BandWidth | `cookbook/sql/finance/bollinger-bands.md`, `.../bollinger-bandwidth.md` |
| RSI, MACD, Stochastic, Rate of change | `cookbook/sql/finance/{rsi,macd,stochastic,rate-of-change}.md` |
| ATR, Donchian, Keltner | `cookbook/sql/finance/{atr,donchian-channels,keltner-channels}.md` |
| Rolling stddev, realized volatility | `cookbook/sql/finance/{rolling-stddev,realized-volatility}.md` |
| OBV, volume profile, volume spikes | `cookbook/sql/finance/{obv,volume-profile,volume-spike}.md` |
| Aggressor volume imbalance | `cookbook/sql/finance/aggressor-volume-imbalance.md` |
| Order flow imbalance (OFI, Cont-Kukanov-Stoikov) | `cookbook/sql/finance/order-flow-imbalance.md` |
| VPIN | `cookbook/sql/finance/vpin.md` |
| Bid-ask spread, liquidity comparison (L2) | `cookbook/sql/finance/{bid-ask-spread,liquidity-comparison}.md` |
| Gamma scalping signal | `cookbook/sql/finance/gamma-scalping-signal.md` |
| Maximum drawdown | `cookbook/sql/finance/maximum-drawdown.md` |
| TICK and TRIN breadth | `cookbook/sql/finance/tick-trin.md` |
| Log returns, cumulative product, compound interest | `cookbook/sql/finance/{log-returns,cumulative-product,compound-interest}.md` |

## Time-Series Patterns

| Recipe | Path |
|--------|------|
| Check timestamp order (table, CSV, Parquet) | `cookbook/sql/time-series/check-timestamp-order.md` |
| Elapsed time between rows | `cookbook/sql/time-series/elapsed-time.md` |
| Force a designated timestamp | `cookbook/sql/time-series/force-designated-timestamp.md` |
| Latest N per partition | `cookbook/sql/time-series/latest-n-per-partition.md` |
| Last N minutes of activity | `cookbook/sql/time-series/latest-activity-window.md` |
| Sessions and elapsed time | `cookbook/sql/time-series/session-windows.md` |
| Filter by week number | `cookbook/sql/time-series/filter-by-week.md` |
| Distribute discrete values across intervals | `cookbook/sql/time-series/distribute-discrete-values.md` |
| Epoch timestamps | `cookbook/sql/time-series/epoch-timestamps.md` |
| Right interval bound with SAMPLE BY | `cookbook/sql/time-series/sample-by-interval-bounds.md` |
| Remove outliers from candles | `cookbook/sql/time-series/remove-outliers.md` |
| FILL from another column (`FILL(PREV(col))`) | `cookbook/sql/time-series/fill-from-one-column.md` |
| FILL PREV with history before the range | `cookbook/sql/time-series/fill-prev-with-history.md` |
| FILL on keyed queries, arbitrary intervals | `cookbook/sql/time-series/fill-keyed-arbitrary-interval.md` |
| Join strategies for sparse sensor data | `cookbook/sql/time-series/sparse-sensor-data.md` |

## Advanced SQL

| Recipe | Path |
|--------|------|
| Subtotals and grand totals (ROLLUP, CUBE, GROUPING) | `cookbook/sql/advanced/subtotals-and-grand-totals.md` |
| Row-level security with views and RBAC | `cookbook/sql/advanced/row-level-security.md` |
| Sankey and funnel diagrams | `cookbook/sql/advanced/sankey-funnel.md` |
| Check a column is sorted by another | `cookbook/sql/advanced/check-column-sort-order.md` |
| Rows before/after a match | `cookbook/sql/advanced/rows-before-after-value-match.md` |
| Local min/max | `cookbook/sql/advanced/local-min-max.md` |
| Top N plus "others" | `cookbook/sql/advanced/top-n-plus-others.md` |
| Pivot with "others" | `cookbook/sql/advanced/pivot-with-others.md` |
| Unpivot (UNION ALL) | `cookbook/sql/advanced/unpivot-table.md` |
| Conditional aggregates | `cookbook/sql/advanced/conditional-aggregates.md` |
| General plus sampled aggregates | `cookbook/sql/advanced/general-and-sampled-aggregates.md` |
| Consistent histogram buckets | `cookbook/sql/advanced/consistent-histogram-buckets.md` |
| Arrays from string literals | `cookbook/sql/advanced/array-from-string.md` |

## Operations

| Recipe | Path |
|--------|------|
| Store QuestDB metrics in QuestDB | `cookbook/operations/store-questdb-metrics.md` |
| Query time histogram (`percentile_disc`) | `cookbook/operations/query-times-histogram.md` |
| Check a transaction was applied | `cookbook/operations/check-transaction-applied.md` |
| Copy data / schema between instances (`SHOW CREATE DATABASE`) | `cookbook/operations/{copy-data-between-instances,copy-schema-between-instances}.md` |
| Many tables: disk and memory | `cookbook/operations/optimize-many-tables.md` |
| Show non-default parameters | `cookbook/operations/show-non-default-params.md` |
| Docker Compose configuration | `cookbook/operations/docker-compose-config.md` |
| CSV import with millisecond timestamps | `cookbook/operations/csv-import-milliseconds.md` |
| TLS with PgBouncer | `cookbook/operations/tls-pgbouncer.md` |

## Integrations

| Recipe | Path |
|--------|------|
| Grafana time range in DECLARE | `cookbook/integrations/grafana/declare-time-range.md` |
| Grafana: dynamic table queries, overlay with time shift, read-only user, name/value dropdown | `cookbook/integrations/grafana/{dynamic-table-queries,overlay-timeshift,read-only-user,variable-dropdown}.md` |
| OPC-UA with Telegraf (dense format) | `cookbook/integrations/opcua-dense-format.md` |
| TLS certificate authorities for clients | `cookbook/programmatic/tls-ca-configuration.md` |

## Demo Dataset

Schema reference: `cookbook/demo-data-schema.md`. Main tables on
`demo.questdb.io`:
- `fx_trades` (timestamp `TIMESTAMP_NS`, symbol, ecn, trade_id, side, passive, price, quantity, counterparty, order_id)
- `market_data` (timestamp, symbol, `bids`/`asks` `DOUBLE[][]`, best_bid, best_ask)
- `core_price` (timestamp, symbol, ecn, bid_price, bid_volume, ask_price, ask_volume, reason, ...)
- Views: `bbo_1s/1m/1h/1d`, `market_data_ohlc_1m/15m/1d`, `fx_trades_ohlc_1m/1d`, `core_price_1s/1d`
- `trades` (crypto) and `trades_OHLC_15m`, `trades_latest_1d`

## Related Reference Pages

`query/sql/{horizon-join,window-join,lateral-join,asof-join,pivot,unnest,sample-by,latest-on,grouping-sets,subsample,declare}.md`,
`query/operators/tick.md`, `query/functions/window-functions/reference.md`.
