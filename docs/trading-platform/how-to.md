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

**2. Load the list of tradable instruments.**

The system needs to know what NIFTY, BANKNIFTY, or any other
stock/option/future actually *is* (its exact price step, lot size,
expiry, etc.) before it can trade or backtest it. This comes from your
broker, not from this project, so it needs to be downloaded once (and
refreshed occasionally, since options/futures expire and new ones get
listed):

```powershell
python scripts/download_instruments.py NFO
```

`NFO` is the exchange segment for futures & options (NIFTY/BANKNIFTY
options and futures live here). If you also want to trade/backtest a
plain stock or an ETF, also run it for the cash market:

```powershell
python scripts/download_instruments.py NSE
```

This is safe to re-run any time — it only adds new instruments and
updates existing ones, it never deletes your trade history.

> **Good to know:** the raw index itself (e.g. literally "NIFTY 50") is
> *not* something you can buy, sell, or backtest directly — only its
> futures, options, or an ETF that tracks it (e.g. `NIFTYBEES`) are
> real, tradable instruments. If you want to analyze "the index", use
> one of those as a stand-in. See the worked example below.

## How to run a backtest

A backtest replays **past** price data through the exact same
strategy/risk/order logic the live system uses, so you can see how a
strategy would have performed — without risking any real money and
without needing the market to be open.

### Step 1 — find the instrument token to test

Every instrument has a numeric "token" the system uses internally. The
easiest way to find one is to ask your already-downloaded instrument
list. For example, to find the NIFTY 50 ETF's token:

```powershell
python -c "from config.settings import get_settings; from trading_system.storage.postgres import PostgresClient; from trading_system.storage.db.repositories import InstrumentRepository; s=get_settings(); pc=PostgresClient(s.postgres_dsn); session=pc.session().__enter__(); repo=InstrumentRepository(session); r=repo.get_by_token(2707457); print(r.instrument_token, r.tradingsymbol, r.exchange, r.segment, r.instrument_type.value, r.lot_size, r.tick_size)"
```

Or, more simply, just reuse the known tokens below — they rarely change:

| What you want to test | Token | Symbol | Exchange/Segment | Lot size |
| --- | --- | --- | --- | --- |
| NIFTY 50 index (via its ETF proxy) | `2707457` | `NIFTYBEES` | `NSE` | 1 |
| NIFTY near-month futures | *(changes monthly — see [Flow & Worked Examples](flow-and-examples.md) for how this was found)* | `NIFTYxxFUT` | `NFO` / `NFO-FUT` | 65 |

> **Why the ETF instead of the raw index or futures?** Your risk settings
> (see below) limit how much money goes into a single trade. One NIFTY
> futures lot is worth well over ₹10 lakh — far more than most configured
> limits allow, so a futures backtest usually rejects every trade with
> `INSUFFICIENT_CAPITAL`. An ETF like `NIFTYBEES` tracks the same index
> but trades in much smaller, affordable units, so it's the practical
> way to test "the index" under realistic account sizes.

### Step 2 — pick your date range and run it

```powershell
python scripts/backtest.py --strategy technical_scoring `
  --instrument-token 2707457 --underlying NIFTY --tradingsymbol NIFTYBEES `
  --exchange NSE --segment NSE --instrument-type EQ --lot-size 1 --tick-size 0.01 `
  --interval-minutes 5 --start 2026-09-28 --end 2026-10-03 --initial-capital 100000
```

What each option means:

| Option | What it controls |
| --- | --- |
| `--strategy` | Which decision-making logic to test. `technical_scoring` is the ready-to-use generic strategy; `no_op` never trades (useful only for testing the pipeline itself). |
| `--instrument-token`, `--underlying`, `--tradingsymbol` | Which instrument to test, found in Step 1. |
| `--exchange`, `--segment`, `--instrument-type`, `--lot-size`, `--tick-size` | The instrument's own details — copy these from the same lookup as the token. |
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
  (bought and then sold again). This system doesn't yet have an
  automatic "exit" feature (see [Safety Mechanisms & Roadmap](safety-and-limitations.md)),
  so a position that's still open at the end of the backtest shows up in
  `net_pnl`/`final_equity` but **not** in `total_trades`. Don't be
  surprised to see `total_trades: 0` alongside a nonzero `net_pnl` — that
  means a trade is open and currently profitable/at a loss, not that
  nothing happened.

For a complete, real worked example with every number explained, see
[Flow & Worked Examples](flow-and-examples.md#6-worked-example-running-a-backtest).

### Backtest troubleshooting

| What you see | What it means | What to do |
| --- | --- | --- |
| `INSUFFICIENT_CAPITAL` rejections | The position size this trade needs is bigger than your configured risk/exposure limits allow. | Test a smaller-notional instrument (an ETF instead of futures), or raise `RISK_MAX_EXPOSURE`/`RISK_MAX_RISK_PER_TRADE` (see below). |
| `OUTSIDE_TRADING_WINDOW` on every signal | Shouldn't happen anymore — `backtest.py` automatically ignores your live trading-hours settings since a backtest replays a past date, not right now. If you still see this, your copy of the script may be out of date. | Pull the latest `scripts/backtest.py`. |
| Zero signals the whole run (`no_signal` every time) | The strategy genuinely never found a setup worth acting on in that date range. This is a normal, valid result. | Try a longer date range, a different instrument, or a different `--interval-minutes`. |
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

Every setting the system reads — your broker login, database/cache
connection, every risk limit, which instruments to watch, logging
level — comes from **one place**, in this order of preference:

1. **AWS Secrets Manager** (one secret, `victor/zerodha/kite` by
   default) — if a value is present here, it wins. This is meant for a
   shared/running deployment, where you want to change a setting without
   editing a file or restarting a server manually.
2. **Your local `.env` file** — used for anything not found in the
   secret, and the easiest way to make a change on your own machine.

**Either way, changes only take effect the next time the application
starts** — nothing is reloaded automatically while it's running.

### Settings you'll most commonly want to change

| What you want to do | Setting name | Example value |
| --- | --- | --- |
| Limit how much you can lose on one trade | `RISK_MAX_RISK_PER_TRADE` | `1000` (rupees) |
| Limit total money committed across all open trades | `RISK_MAX_EXPOSURE` | `200000` (rupees) |
| Limit how many trades happen in a day | `RISK_MAX_DAILY_TRADES` | `10` |
| Stop trading for the day after a loss | `RISK_MAX_DAILY_LOSS` | `5000` (rupees) |
| Only trade during certain hours | `RISK_TRADING_WINDOW_START` / `RISK_TRADING_WINDOW_END` | `09:20` / `15:00` |
| Avoid the volatile first/last few minutes of the session | `RISK_AVOID_FIRST_MINUTES_OF_SESSION` / `RISK_AVOID_LAST_MINUTES_OF_SESSION` | `15` |
| Choose which indices/stocks to trade as options | `OPTIONS_WATCHLIST` | `NIFTY,BANKNIFTY` |
| Choose which stocks to trade directly (no options) | `EQUITY_WATCHLIST` | `RELIANCE,TCS` |
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
