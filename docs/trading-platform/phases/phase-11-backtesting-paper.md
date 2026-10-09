# Phase 11 — Backtesting & Paper Trading Engine

Phase 11 evaluates whatever strategy is configured (no real strategy
specification exists yet -- see Phase 7A -- so this works generically,
the same way `StrategyEngine` itself does) against historical data
(backtest) or live data (paper trading) **without ever placing a real
broker order**. It reuses the real, unmodified Phase 7 `StrategyEngine`,
Phase 8 `RiskEngine`, and Phase 9 `OrderExecutionEngine` -- not a second
implementation of strategy/risk/execution logic.

```text
Historical candles (backtest) or live ticks (paper)
  -> MarketDataService (Phase 6, unmodified)
  -> StrategyEngine.evaluate_candle() (Phase 7, unmodified)
  -> RiskEngine.evaluate_signal() (Phase 8, unmodified)
  -> OrderExecutionEngine.execute()/refresh_status() (Phase 9, unmodified)
       -> BacktestExecutionAdapter / PaperExecutionAdapter (Phase 11 only;
          never KiteBrokerAdapter, never a real Kite order-placement call)
  -> Portfolio (Decimal cash/position/P&L/equity/drawdown accounting)
  -> BacktestResult / paper-session summary + PostgreSQL audit trail
```

**A pre-existing bug this phase surfaced and fixed:** wiring the real
end-to-end pipeline together for the first time revealed that
`StrategyEngine._validate_and_emit()` never persisted the in-memory
`Signal.id` onto its own `SignalRepository` row -- every real signal's
persisted ID silently differed from the ID `RiskEngine`/`OrderExecutionEngine`
later looked up, which would have made **every** real (non-test-fixture)
signal fail Phase 9 validation with `SIGNAL_NOT_FOUND`. Fixed by passing
`id=signal.id` through (one line); this is a correctness fix, not a
strategy/risk change, and is covered by the new `test_backtest_engine.py`
end-to-end test that would otherwise fail.

**Simulated-clock additions (additive, zero behavior change for
Phases 1-10):** `MarketState`, `StrategyEngine`, `PostgresAccountStateProvider`,
and `OrderExecutionEngine` each gained an optional `now_fn`/`today_fn`
constructor parameter (default: real wall-clock, exactly as before).
Without this, replaying historical dates against the *real* engines would
silently see every tick as stale and every historical option as "expired
today" relative to the real current date -- never because any rule
changed, only because "now" was previously always assumed to be the real
wall clock. `RiskEngine._check_trading_window()`'s real-wall-clock IST
check (`trading_window_start`/`_end`, `avoid_first/last_minutes_of_session`)
and Redis-TTL-based cooldown (`RiskPolicy.cooldown_seconds`) were
deliberately **not** clock-injected (see Known Limitations) -- leave them
unset for a backtest.

**Isolation:** every backtest run gets its own ephemeral SQLite database
(`data/backtests/{run_id}.sqlite`, created fresh via
`Base.metadata.create_all`) and its own in-memory `fakeredis` instance --
the simulated `Signal`/`RiskDecision`/`TradeIntent`/`Order`/`ExecutionAttempt`
rows Phase 7/8/9 persist internally live **only** there, never in the real
production database. Only the run's configuration snapshot and final
result/trade summary are durably recorded in the real database's new
`backtest_runs`/`backtest_trades` tables. Paper trading, by contrast,
reuses the real shared PostgreSQL/Redis (already environment-namespaced by
`RedisKeyBuilder`/`app_env` since Phase 3/9) -- a new `paper_sessions`
table tracks session lifecycle; simulated paper orders still go through
the existing Phase 9 safety gate (a non-live adapter requires a
non-production `APP_ENV`), so they can never be mistaken for/escalate into
a real order.

**Historical data (`backtest/data_loader.py`):** `HistoricalDataLoader`
prefers already-persisted PostgreSQL candles
(`MarketDataService.get_candles`) and falls back to the existing Kite
historical endpoint (`HistoricalMarketDataProvider`, Phase 6) -- never a
second Kite client. Validates ordering, duplicate timestamps, missing
intervals, and invalid OHLC; an invalid/duplicate/out-of-order candle is
excluded (never corrected/fabricated) and reported as a
`DataQualityWarning`, not silently dropped.

**Look-ahead-bias protection:** a signal generated from candle `i`'s
close can only ever fill at candle `i+1`'s open (or later, for a
limit/SL order) -- `BacktestExecutionAdapter` tracks the candle a request
was submitted on and refuses to resolve a fill until a *later* candle is
current. `test_backtest_engine.py::test_backtest_does_not_use_future_candles_for_earlier_decisions`
proves that replacing every candle after the signal bar with different
(wildly different) prices never changes the strategy's own decision
sequence up to that bar.

**Simulated execution assumptions (`backtest/execution_adapter.py`):**
MARKET orders fill at the *next* candle's open, adjusted by configured
`slippage_bps` in the adverse direction (BUY pays more, SELL receives
less); if no later candle ever arrives, the order is reported `CANCELLED`
at the end of the run, never fabricated as filled. LIMIT/SL orders fill
only once a later candle's high/low actually reaches the limit/trigger
price ("touch" rule), at the limit price itself, never better. No
partial fills are modeled (no order-book depth data exists).
`PaperExecutionAdapter` uses the same slippage model against the current
real-time last-traded price instead of "next bar". Both adapters
implement the existing Phase 9 `BrokerOrderAdapter` interface and are
handed to the real `OrderExecutionEngine` -- neither ever calls
`KiteBrokerAdapter`/`Broker.place_order`.

**Transaction costs (`backtest/costs.py`):** `TransactionCostModel` --
every rate (brokerage, exchange charges, STT, GST, SEBI charges, stamp
duty) defaults to **zero** and `verified=False`; nothing invents a "real"
current rate. Supply real rates explicitly and set `verified=True` before
treating a result as realistic.

**Portfolio accounting (`backtest/portfolio.py`):** `Portfolio` tracks
cash, one signed-quantity open position per instrument (positive = long,
negative = short, matching `storage.db.models.Position`'s convention),
realized/unrealized P&L, equity, peak equity, and drawdown, entirely in
`Decimal`. Averaging into a position weighted-averages the entry price;
an opposite-direction fill reduces/closes/reverses it (a fill larger than
the open position closes it fully and opens a new one with the leftover
quantity at the fill price) -- hand-verified in `test_backtest_portfolio.py`.
`equity()` is cash plus the current market value of open positions
(`price * signed_quantity`), never cash plus paper P&L alone (which would
double-count the capital already moved to/from cash at entry).

**Performance metrics (`backtest/metrics.py`):** win rate/profit factor/
average win-loss/consecutive win-loss/average holding time are computed
from closed trades only; a metric that cannot be honestly computed (zero
trades, zero losing trades, fewer than 2 equity-curve return
observations, or zero return variance) is `None`, never a
fabricated/infinite value. Sharpe/Sortino use a documented, overridable
annualization (`periods_per_year=252`) and risk-free rate (default zero,
never a guessed real rate).

**Known limitation -- no exit/position-closing pipeline exists yet:**
inspecting Phases 7-9 confirms `RiskEngine._check_signal()` rejects any
non-BUY/SELL `Signal.signal_type` (including `EXIT`) as "not a
risk-manageable entry" -- there has never been a mechanism anywhere in
this project for a strategy to close a position it opened, nor anything
that monitors an open position's stop-loss/target against ongoing price
movement. Phase 11 does not invent one (per its own "do not implement a
second version of the risk engine" boundary) -- a backtested/paper
position can only ever be opened; it remains open (marked-to-market, never
force-closed or fabricated-closed) until the run ends. This is an accurate
simulation of what the real system would do today, not a Phase 11 gap --
see [Safety Mechanisms & Roadmap](../safety-and-limitations.md) for Phase 12 TODOs.

**Paper-session management (`backtest/paper.py`):** `PaperTradingSession`
is keyed by `strategy_name:instrument_token:timeframe`; `start()` rejects
a second concurrent `RUNNING` session for the same key
(`PaperSessionRepository.get_running_by_session_key`). A stopped session's
candle listener checks its own status and drops any further event --
proven in `test_paper_trading_session.py`. There is no background
daemon/process manager in this phase; `scripts/paper_trade.py` runs in the
foreground until `Ctrl+C`.

**Restart/recovery:** `BacktestRunRepository.list_incomplete()`
finds runs that never reached `COMPLETED`/`FAILED` (e.g. the process was
killed mid-run) for manual inspection -- Phase 11 does not auto-resume a
backtest or auto-resume live execution. A stopped/crashed paper session's
`last_processed_timestamp`/`summary_metrics` are durably recorded on
`stop()`; resuming reads only paper-specific durable state, never assumes
Redis survived, and never auto-enables live trading.

**CLI:** `scripts/backtest.py` (`--strategy no_op|technical_scoring
--instrument-token ... --start YYYY-MM-DD --end YYYY-MM-DD
--initial-capital ...`) and `scripts/paper_trade.py start|status`. No web
framework was added.

**Persistence/migration:** `0010_backtest_paper` adds `backtest_runs`,
`backtest_trades`, `paper_sessions`. Apply `alembic upgrade head` before
using Phase 11.

**Tests and boundary:** `test_backtest_portfolio.py` (9, hand-computed
accounting examples), `test_backtest_metrics.py` (9, zero-trade/zero-loss/
insufficient-data safety), `test_backtest_costs.py` (4),
`test_backtest_execution_adapter.py` (6, including the next-bar-fill and
touch-rule limit-fill assertions), `test_backtest_data_loader.py` (6),
`test_backtest_engine.py` (5, end-to-end including the look-ahead-bias
proof and a never-calls-real-broker safety assertion), and
`test_paper_trading_session.py` (4) -- all SQLite/fakeredis/fake-data
only. The gated `tests/integration/test_backtest_integration.py` persists
a real `backtest_runs`/`backtest_trades` row against real PostgreSQL (fake
historical data only) and cleans up after itself. All Phase 1-10 tests
confirmed still passing (561 total after Phase 11).

## Fixes discovered running this against real PostgreSQL for the first time

The full test suite runs against in-memory SQLite, which is lenient about
things real PostgreSQL enforces strictly. The first time this phase was
actually run end-to-end against a real database, three issues surfaced
that no amount of unit testing had caught:

- **Pending migrations.** `alembic upgrade head` had never actually been
  applied beyond migration `0005` in the real database -- always check
  `alembic current` vs `alembic heads` before assuming the schema is up
  to date.
- **A migration bug** (`migrations/versions/0006_risk_trade_intents.py`):
  the `trade_intent_direction` enum type was created twice in the same
  migration (once explicitly, once implicitly via the column definition),
  which PostgreSQL rejects outright (SQLite silently allows it). Fixed by
  removing the redundant explicit creation.
- **A stop-loss/target precision bug** (`risk/stop_loss.py`): ATR-derived
  stop-loss/target values carry many decimal places, but the database
  columns that store them only keep 2. PostgreSQL silently rounds on
  insert; SQLite does not enforce this at all. The mismatch between the
  in-memory and re-read values caused Phase 9 to reject every single
  approved trade with `STOP_LOSS_MISMATCH`. Fixed by rounding stop-loss
  and target to 2 decimal places at the moment they're calculated.

See the [How-To Guide](../how-to.md) for the practical, step-by-step
version of running a backtest or paper session without hitting these.

## Addendum — dashboard and editable settings (frontend + backend API)

This wasn't part of the original Phase 11 prompt -- it's a separate,
external idea the project owner asked for afterward ("let me actually
*see* this data"), folded into Phase 11 here rather than given its own
phase number, since "Phase 12" was already reserved in the original
roadmap for a later, unrelated "Production & Live-Trading Readiness"
phase (see `specs/08-phase8.md`/`specs/09-phase9.md`/`specs/10-phase10.md`).

```text
Browser
  -> GET / (server-rendered HTML, trading_system/api/app.py + Jinja2,
            frontend/templates/index.html)
  -> GET /static/... (CSS + jQuery, frontend/static/)
  -> GET /api/... (FastAPI JSON endpoints, trading_system/api/routers/)
  -> the SAME PostgreSQL database every other phase already writes to
```

**Why this shape:**

- **Trading actions remain out of scope.** The dashboard has no code path
  into `OrderExecutionEngine`, `RiskEngine`, or broker order calls. Its
  only write route is a token-protected settings update; settings are
  allow-listed, validated, persisted in PostgreSQL, and append an audit
  row for each change.
- **Scores and trades remain separate records.** Existing closed-trade
  details are served from `trades`; normalized scoring results are
  stored in the new `score_snapshots` table and linked to their signal.
  `/api/scores` includes the instrument symbol and token.
- **Overview metrics use durable records.** Filled orders, open
  positions, closed-trade P/L, win ratio, and failed orders are queried
  from PostgreSQL. Available account cash is fetched live from Kite's
  equity margins endpoint; it is not represented as a database value and
  is shown unavailable when Kite cannot supply it.
- **Settings apply on process start.** Dashboard overrides are loaded
  before trading components are constructed. They do not hot-reload; the
  trading process must restart. `DASHBOARD_ADMIN_TOKEN` is environment-
  only, and an unset token disables edits. The dashboard's same-origin
  UI sends the token in `X-Dashboard-Token`.
- **The page is rendered in Python, not just served as a static file.**
  `trading_system/api/app.py::dashboard_home()` renders
  `frontend/templates/index.html` through Jinja2
  (`fastapi.templating.Jinja2Templates`), passing a title/version/
  generated-at timestamp from Python into the page -- provable by the
  "Rendered by FastAPI + Jinja2 at ..." footer on every page load. Only
  the CSS and jQuery script are plain static files, served from
  `frontend/static/` at `/static/...`.
- **Plain jQuery, not a frontend framework**, per the explicit
  requirement -- no build step, bundler, or package manager on the
  client side.
- **Generic over hand-written per-table schemas.** `api/serialization.py::row_to_dict()`
  converts any ORM row's mapped columns into a JSON-safe dict generically
  (via `sqlalchemy.inspect()` + FastAPI's own `jsonable_encoder`) instead
  of maintaining 15+ near-duplicate response schemas by hand. The
  frontend mirrors this: `renderTable()` in `app.js` builds a table's
  columns from whatever keys a response actually contains, so a new
  field or endpoint never requires a frontend code change.

**What's exposed:** health (`/api/health`), instruments + candles,
signals/orders/positions/trades (each enriched with its instrument's
`tradingsymbol`/token via `api/queries.py::instrument_extras_by_row_id()`
-- never a bare instrument UUID), risk decisions + trade intents,
reconciliation runs + their order/position mismatches, and backtest runs
+ their trades + paper sessions. The dashboard addendum also exposes
`/api/scores`, `/api/dashboard/metrics`, and settings GET/PUT endpoints.
The PUT endpoint requires `DASHBOARD_ADMIN_TOKEN`; setting overrides and
audit records are stored in `dashboard_settings` and
`dashboard_setting_audit`. Every list is most-recent-first and bounded by
a `limit` (`api/queries.py::MAX_LIMIT`, 500).

**Run it:** `python scripts/run_dashboard.py`, then open
`http://127.0.0.1:8000/`. See the
[How-To Guide](../how-to.md#how-to-view-the-dashboard) for the practical
walkthrough.

**Testing note:** FastAPI's `TestClient` executes a request on a
different thread than the test itself. The shared
`tests/unit/conftest.py::db_session` fixture's SQLite engine is bound to
a single thread by default, so `tests/unit/test_api.py` uses its own
engine/session fixture with `StaticPool` + `check_same_thread=False`
instead. API tests cover health, trading records, normalized score
snapshots, KPI calculations, token-protected settings writes, validation,
audit rows, and runtime override precedence, along with the existing
backtest/reconciliation/dashboard routes. The current migration has not
been applied to a real database as part of this addendum; apply
`alembic upgrade head` before using the new score/settings tables. The
full unit suite currently passes with 577 tests.

See the original prompt: [specs/11-phase11.md](../specs/11-phase11.md).
