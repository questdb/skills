# QuestDB skill for coding agents

Teaches coding agents (Claude Code, Codex and others) to write correct QuestDB
SQL and client code, and to build live dashboards in the QuestDB Web Console.
Last updated for QuestDB 10.2 and QuestDB Enterprise 4.2. See the
[release notes](https://questdb.com/release-notes/) and the
[docs changelog](https://questdb.com/docs/changelog/) for later changes.

## Installation

### npx (recommended)

```shell
npx skills add questdb/skills
```

This installs the skill globally for Claude Code. To install it for a specific
project only, run the command from the project directory with the --local flag.

#### Manual copy

Copy the questdb/ folder from this repository into any of these locations so
your agent picks it up:

##### Claude Code:
- ~/.claude/skills/questdb/ - available in all projects
- <project>/.claude/skills/questdb/ - available in a specific project

##### Codex:
- ~/.codex/skills/questdb/ - available in all projects
- <project>/.codex/skills/questdb/ - available in a specific project

The folder must contain SKILL.md and the references/ directory.

### Dashboards: the QuestDB MCP server

To let the agent build dashboards and notebooks in the Web Console, also
install the QuestDB MCP server and keep the Web Console open in a browser:

```shell
npx @questdb/mcp-server-questdb@latest setup
```

## What the skill covers

- QuestDB SQL: TICK time filters, SAMPLE BY, LATEST ON, DECLARE variables and
  value lists, ASOF / HORIZON / WINDOW / LATERAL joins, window functions,
  GROUPING SETS, SUBSAMPLE, PIVOT, UNNEST, arrays.
- Schema design: designated timestamps, partitioning, DEDUP UPSERT and SKIP,
  TTL, Parquet partitions, indexes.
- Materialized views (including passthrough views and EXPIRE ROWS), live
  views, and regular views with parameters.
- Ingestion and queries with the QWP clients (Python, Java, Go, Rust, C/C++,
  .NET, Node.js, Ruby, browser), plus REST, PGWire and ILP.
- Web Console dashboards through the MCP server, with two complete reference
  dashboards (a live FX order book and an instance monitoring dashboard).
- Monitoring with `sys.metrics` and WAL telemetry.
- Enterprise: RBAC, tokens, failover, storage policies and cold storage.
- Finance recipes: indicators, execution quality (markout, slippage,
  implementation shortfall), and an index of the official cookbook.
- Grafana, when you ask for it.

## Try it out

With QuestDB running locally and the MCP server installed:

```
Build a real-time crypto market data pipeline using cryptofeed (OKX exchange) ingesting trades and L2 order
book data into QuestDB. Symbols: BTC-USDT,ETH-USDT,SOL-USDT,XRP-USDT,ADA-USDT,AVAX-USDT,DOT-USDT,LINK-USDT.

Then build a live dashboard in the QuestDB Web Console with OHLC candlesticks and volume, VWAP, Bollinger
Bands, RSI, a market depth chart and a spread panel, with a symbol variable.

Do this in python. Create a venv in the current folder and install the bare minimum you need.
```

If you don't have QuestDB running, add "start QuestDB with docker (latest image)" to the prompt.

To use Grafana instead, ask for it explicitly ("...then create a Grafana dashboard...") and say whether
Grafana is already running.

Other things to try:

```
Using the QuestDB demo dataset, build a notebook that compares markouts per ECN at 1s, 5s, 30s and 1m.
```

```
Build a monitoring dashboard for my QuestDB instance from sys.metrics.
```

```
Keep only the latest reading per sensor in a materialized view, and a rolling 20-row Bollinger band
as a live view.
```

You can follow up with:

```
what other indicators can I add to the dashboard for this dataset?
```

When done, if you want to remove everything, you can ask:

```
please, stop ingestion, remove the containers, views, tables, notebooks and local files you created in
this session, so we can restart from scratch
```
