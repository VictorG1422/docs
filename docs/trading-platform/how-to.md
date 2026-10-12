# How-To Guide

This page is a practical, step-by-step guide to actually *using* the
trading system — running a backtest, running it against live market data
in safe paper mode, changing settings, and the handful of everyday tasks
around those. It's written for someone operating the system day to day,
not for someone reading the source code.

If you haven't installed/configured the project yet, do that first: see
[Setup & Configuration](setup.md). If you want to understand *how* the
system makes a decision before you start changing things, read
[Flow & Worked Examples](flow-and-examples.md) — it walks through a full
example trade with real numbers.

Every command below is run from the project folder (`trading_platform`)
with your virtual environment activated:

```powershell
cd trading_platform
.\.venv\Scripts\Activate.ps1
```

## Before you start: two one-time steps

**1. Check that everything is connected.**

```powershell
python scripts/health_check.py
```

This tells you whether the database, the cache, and your broker
connection are all working. Fix any failures here before doing anything
else — nothing downstream will work reliably otherwise.

**2. Load the instrument master.**

The system needs broker metadata for contracts and NSE index references.
Download the instrument master once and refresh it occasionally as new
contracts are listed:

```powershell
python scripts/download_instruments.py
```

With no arguments, the script syncs both `NFO` (futures/options) and
`NSE` (cash-market instruments and index references). You can request one
exchange or both explicitly:

```powershell
python scripts/download_instruments.py NSE
python scripts/download_instruments.py NFO
python scripts/download_instruments.py NSE NFO
```

The sync upserts by instrument token, so it is safe to re-run and does not
delete existing instrument or trade rows.

The main trading application, paper-trading start command, and backtest
command also refresh **both NSE and NFO at every process start** before
trading or historical-data work proceeds. A Kite download or database
sync failure aborts that process; inspect the exchange-specific error and
restore connectivity before retrying. The dashboard server is read-only
with respect to instrument sync and does not trigger this startup refresh.
The manual command remains useful for seeding the instrument table before
a token-watchlist migration or dashboard configuration.

> **Index references:** the NSE dump includes `NIFTY 50` and `NIFTY BANK`
> (the BANKNIFTY index). They are stored for inspection, with Kite's
> `INDICES` segment and zero lot/tick sizes; they are not directly tradable.
> For an options-watchlist underlying, use the index reference as the signal
> instrument and let the system select/trade its CE/PE. Equity-watchlist entries
> must be tradable cash equities; an index cannot be directly ordered. In the dashboard, search for
> `NIFTY BANK` rather than `BANKNIFTY`, which is the familiar name but not
> Kite's index-master symbol.

The watchlist picker stores the selected underlying instrument token (not
the displayed name), so search and select the synchronized EQ row after
instrument sync. Existing databases with symbol-based watchlists must run
`python scripts/migrate_watchlists_to_tokens.py` and review the dry-run
before `--apply` and before starting the token-based application version.

## How to view the dashboard

Before running a backtest or paper session blind, it's worth knowing you
can *see* the data directly in a browser instead of reading raw log
lines or querying the database by hand:

```powershell
python scripts/run_dashboard.py
```

Open `http://127.0.0.1:8000/` in any browser. The dashboard reads its
trading records from PostgreSQL and has a protected editor for the
allow-listed risk and scoring settings. It cannot place orders or approve
trades. Use the shared sidebar to open independent, bookmarkable pages:

- **Dashboard** (`/`) — filled trades, open positions, realized and unrealized
  P/L, win ratio, failed/pending orders, and activity counts. Available
  account cash is fetched live from Kite and shown as unavailable if the
  broker balance cannot be retrieved.
- **Metrics** (`/metrics`) — the complete set of account, activity, and
  P/L metrics.
- **Score history** (`/scores`) — normalized score snapshots stored separately from signals
  and trades, including instrument symbols and scoring matches.
- **Score settings** (`/score-settings`) — scoring thresholds and
  individual pattern/indicator weights.
- **Instruments & watchlists** (`/instruments`) — search synchronized
  instrument records and add/remove token-based options/equity watchlist
  entries. Options watchlists analyze the underlying EQ/index and trade
  the selected CE/PE; equity watchlists analyze and trade the same cash EQ.
- **Backtests** (`/backtests`) — inspect run metrics and each completed
  trade. Run details include equity, gross/net P&L, win/loss counts,
  drawdown, holding time, Sharpe/Sortino, streaks, costs, exposure, and
  rejected signals. The audit table expands score components into separate
  timeframe, direction, contribution, strength, weight, multiplier, and
  detail columns; every risk check has its own status and reason columns.
- **Trading & risk settings** (`/risk-settings`) — trading mode, risk
  limits, trading hours, and stop-loss/target parameters.
- **Operational pages** — Signals, Orders, Positions, Trades, Risk
  Decisions, Trade Intents, Reconciliation, Candles, and Paper Sessions
  each have their own page and refresh action.

Every displayed date/time is explicitly formatted in **IST
(`Asia/Kolkata`)**, regardless of the browser or computer timezone.

To enable editing, configure `DASHBOARD_ADMIN_TOKEN` in AWS Secrets
Manager and restart the dashboard process. Enter that token in the
settings section before saving. Keep it secret: it authorizes changes to
trading risk and scoring configuration. Editing remains disabled when
the token is empty. Apply the database migrations first with `alembic upgrade head`;
migration `0011` adds the score/settings tables and `0012` permits NSE
index reference rows.

The three colored dots next to the logo show whether PostgreSQL, Redis,
and your Kite connection are currently healthy (the same check as
`scripts/health_check.py`) and refresh automatically every 20 seconds.

Stop it with `Ctrl+C` like any other script here — it holds no state of
its own and is always safe to stop and restart.

## How to run a backtest

A backtest replays **past** price data through the exact same
strategy/risk/order logic the live system uses, so you can see how a
strategy would have performed — without risking any real money and
without needing the market to be open.

### Step 1 — find the instrument token to test

Every instrument has a numeric token the system uses internally. In options
mode, use the synchronized cash/index row as the signal source; trade sizing
uses the selected option's own lot size. To inspect the NIFTY 50 index row:

```powershell
python -c "from sqlalchemy import select; from config.settings import get_settings; from trading_system.storage.postgres import PostgresClient; from trading_system.storage.db.models import Instrument; s=get_settings(); pc=PostgresClient(s.postgres_dsn); session=pc.session().__enter__(); r=session.scalar(select(Instrument).where(Instrument.segment == 'INDICES', Instrument.tradingsymbol == 'NIFTY 50')); print(r.instrument_token, r.tradingsymbol, r.exchange, r.segment, r.instrument_type.value, r.lot_size, r.tick_size)"
```

Verify the token against the synchronized database:

| What you want to test | Token | Symbol | Exchange/Segment | Lot size |
| --- | --- | --- | --- | --- |
| NIFTY 50 index signal source | `256265` | `NIFTY 50` | `NSE` / `INDICES` | `0` in the instrument master |
| NIFTY near-month futures | *(changes monthly; not an index signal source)* | `NIFTYxxFUT` | `NFO` / `NFO-FUT` | varies |

### Step 2 — pick your date range and run it

```powershell
python scripts/backtest.py --strategy technical_scoring `
  --instrument-token 256265 --underlying "NIFTY 50" --tradingsymbol "NIFTY 50" `
  --exchange NSE --segment INDICES --instrument-type EQ --lot-size 1 --tick-size 0.05 `
  --interval-minutes 5 --start 2026-09-28 --end 2026-10-03 --initial-capital 100000
```

What each option means:

| Option | What it controls |
| --- | --- |
| `--strategy` | Which decision-making logic to test. `technical_scoring` is the ready-to-use generic strategy; `no_op` never trades (useful only for testing the pipeline itself). |
| `--instrument-token`, `--underlying`, `--tradingsymbol` | The underlying cash/index instrument whose candles drive signal generation. |
| `--exchange`, `--segment`, `--instrument-type`, `--lot-size`, `--tick-size` | Signal-instrument metadata. In OPTIONS mode the selected contract has its own execution metadata and lot size. |
| `--interval-minutes` | The candle size the strategy reacts to (5-minute candles here). |
| `--start` / `--end` | The historical date range, `YYYY-MM-DD`. |
| `--initial-capital` | The pretend starting balance, in rupees. |

### Step 3 — read the result

The last line printed tells you everything:

```json
{"event": "Backtest finished", "status": "COMPLETED", "total_trades": 0, "net_pnl": "1840.0000", "final_equity": "101840.0000", "errors": []}
```

- **`status: COMPLETED`** — the run finished properly. (`FAILED` means
  something broke — check the `errors` list.)
- **`final_equity`** — what your pretend account is worth at the end.
- **`net_pnl`** — final_equity minus your starting capital. This is the
  number you actually care about.
- **`total_trades`** — only counts trades that were **fully closed**
  (bought and then sold again). Stop-loss/target exits are simulated from
  the traded instrument's candles, but a position still open at the end
  of the backtest shows up in
  `net_pnl`/`final_equity` but **not** in `total_trades`. Don't be
  surprised to see `total_trades: 0` alongside a nonzero `net_pnl` — that
  means a trade is open and currently profitable/at a loss, not that
  nothing happened.

For a complete, real worked example with every number explained, see
[Flow & Worked Examples](flow-and-examples.md#6-worked-example-running-a-backtest).

### Backtest troubleshooting

| What you see | What it means | What to do |
| --- | --- | --- |
| `INSUFFICIENT_CAPITAL` rejections | The selected option/equity position exceeds configured risk or exposure limits. | Review the contract and persisted risk checks in the backtest audit before changing limits. |
| `OUTSIDE_TRADING_WINDOW` on every signal | Shouldn't happen anymore — `backtest.py` automatically ignores your live trading-hours settings since a backtest replays a past date, not right now. If you still see this, your copy of the script may be out of date. | Pull the latest `scripts/backtest.py`. |
| Zero trades despite signal activity | Option contract/premium candles may be unavailable, or risk checks may reject signals. | Inspect score-component columns, signal/risk reasons, and data-quality warnings in the run audit. |
| `status: FAILED` | Something broke — most commonly, the database schema isn't up to date. | Run `alembic upgrade head` (see "How to update the database" below), then try again. |

## How to run paper (live) trading

Paper trading is like a backtest, except it watches the **real, live**
market tick by tick instead of replaying history. No real orders are
ever placed — every fill is simulated — but everything else behaves
exactly like the live system would.

**This only works while the market is actually open**: Monday–Friday,
09:15–15:30 IST (and not on NSE holidays).

```powershell
python scripts/paper_trade.py start --strategy technical_scoring `
  --instrument-token 12468226 --underlying NIFTY --tradingsymbol NIFTY26OCTFUT `
  --interval-minutes 5 --initial-capital 100000
```

The flags mean the same thing as for a backtest, minus the date range
(there isn't one — it watches whatever happens from the moment you start
it). It runs in your terminal until you stop it with `Ctrl+C`.

To check on a session you started earlier (even from another terminal):

```powershell
python scripts/paper_trade.py status --session-key technical_scoring_strategy:12468226:5
```

(The session key is just `<strategy>_strategy:<instrument-token>:<interval-minutes>`.)

## How to go live (placing real orders) — read this first

**Live trading is turned off by default, on purpose, and in three
separate places at once.** All three must be explicitly set for a real
order to ever reach your broker:

1. `TRADING_MODE=live`
2. `APP_ENV=production`
3. `ORDER_EXECUTION_ENABLED=true`

This is deliberate: the system assumes paper trading unless you
*unambiguously* say otherwise. Before touching any of these:

- Run your strategy in paper mode for a meaningful stretch of time first
  and review the results honestly.
- Make sure your risk settings (below) reflect money you are actually
  willing to lose — they are the only thing standing between a strategy
  decision and a real order.
- Understand that this system currently has **no automatic exit**: once
  a position opens, nothing closes it for you (see
  [Safety Mechanisms & Roadmap](safety-and-limitations.md)). You are
  responsible for monitoring and manually closing any live position.

Set all three values (see "How to change settings" below), then run the
live application the same way you would in paper mode —
`python -m trading_system.main` starts the full application, though note
that a specific strategy/instrument still needs to be wired in to
actually trade (see [Safety Mechanisms & Roadmap](safety-and-limitations.md#what-remains-to-be-implemented)
for the current state of this).

## How to change settings

Configuration has distinct owners. AWS Secrets Manager
(`victor/zerodha/kite`) supplies the allow-listed broker/Git credentials,
PostgreSQL/Redis connection values, and dashboard admin token.
PostgreSQL dashboard settings supply watchlists, trading mode, risk
controls, and scoring weights/thresholds; saved rows override
corresponding local settings when present. `.env` supplies bootstrap and
local/runtime values such as AWS region/secret name, app environment,
and log level.

Dashboard edits are validated and audited in PostgreSQL. Restart the
trading process for them to take effect; settings are not hot-reloaded.
The dashboard admin token comes from AWS Secrets Manager and is not
stored in PostgreSQL.

### Settings you'll most commonly want to change

| What you want to do | Setting name | Example value |
| --- | --- | --- |
| Limit how much you can lose on one trade | `RISK_MAX_RISK_PER_TRADE` | `1000` (rupees) |
| Limit total money committed across all open trades | `RISK_MAX_EXPOSURE` | `200000` (rupees) |
| Limit how many trades happen in a day | `RISK_MAX_DAILY_TRADES` | `10` |
| Stop trading for the day after a loss | `RISK_MAX_DAILY_LOSS` | `5000` (rupees) |
| Only trade during certain hours | `RISK_TRADING_WINDOW_START` / `RISK_TRADING_WINDOW_END` | `09:20` / `15:00` |
| Avoid the volatile first/last few minutes of the session | `RISK_AVOID_FIRST_MINUTES_OF_SESSION` / `RISK_AVOID_LAST_MINUTES_OF_SESSION` | `15` |
| Choose which indices/stocks to trade as options | `OPTIONS_WATCHLIST` | Select underlying EQ instruments in the dashboard; persisted as token IDs |
| Choose which stocks to trade directly (no options) | `EQUITY_WATCHLIST` | Select tradable cash EQ instruments in the dashboard; persisted as token IDs |
| Switch between simulated and real orders | `TRADING_MODE` | `paper` or `live` |
| See more detail in the logs | `LOG_LEVEL` | `DEBUG` |

The full list of every available setting, with a plain description of
each, is in [Setup & Configuration](setup.md). That page also explains
exactly how the AWS secret works if you're maintaining a shared
deployment.

> **Note on trading-hours settings and backtests:** the trading-window
> and session-edge settings above are compared against the real current
> time, because they exist to protect *live* trading. They're
> automatically ignored during a backtest (which replays a past date) so
> you don't need to change them back and forth — see the backtest
> troubleshooting table above.

## How to update the database (migrations)

The database's structure occasionally needs to catch up with the code
(e.g. a new feature needs a new table). This is a one-time step after
installing or updating the project:

```powershell
alembic upgrade head
```

It's always safe to run — it only adds what's missing and never deletes
your data. If you ever see a run fail for an unexplained reason, check
whether this step was missed:

```powershell
alembic current   # shows what your database thinks it's at
alembic heads      # shows the latest version the code expects
```

If the two don't match, run `alembic upgrade head` again.

## How to check what's happening (logs)

Everything the system does is logged as one line of structured JSON per
event, both to your screen and to `logs/trading_system.log`. A few
events worth knowing how to spot:

| Log event | What it means |
| --- | --- |
| `STRATEGY_EVALUATED` (`result: no_signal`) | The strategy looked at the latest candle and decided there was nothing worth doing — the normal, most common outcome. |
| `RISK_APPROVED` | A trade passed every risk check and is about to be sized and submitted. |
| `RISK_REJECTED` | A trade was turned down — the `reason` field says exactly why (e.g. `INSUFFICIENT_CAPITAL`, `MAX_DAILY_LOSS_EXCEEDED`, `DUPLICATE_POSITION`). |
| `ORDER_FILLED` | A (real or simulated) order was completed. |

Set `LOG_LEVEL=DEBUG` (see "How to change settings" above) if you want
far more detail while diagnosing something.

## Frequently asked questions

**Why did my backtest only open one trade all week, even though I saw
many signals in the log?**
Every signal after the first one got rejected as a duplicate position —
the system doesn't yet have a way to close a position automatically, so
once one is open, nothing else can be opened for that same instrument
until you add an exit mechanism. This is expected, documented behavior,
not a bug — see [Safety Mechanisms & Roadmap](safety-and-limitations.md).

**My backtest/paper session shows a profit or loss, but "total trades"
is zero — is that a bug?**
No — see "Step 3 — read the result" above. The P&L reflects an open
position being marked to the latest price; it only becomes a counted
"trade" once it's closed.

**Can I trade NIFTY/BANKNIFTY itself?**
Not directly — only its futures, its options, or an ETF that tracks it.
See "Before you start" above.

**I changed a setting but nothing happened.**
Settings are only read once, at startup. Stop the running
process/session and start it again.
