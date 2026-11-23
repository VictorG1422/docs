# Safety Mechanisms & Roadmap

## Safety mechanisms

- **The process never exits because the market is closed.** This is the
  single biggest safety change in the most recent upgrade round: the app
  used to shut itself down at NSE close and had to be manually restarted
  every session. It now runs a small state machine (`STARTING` →
  `WAITING_FOR_MARKET` → `WARMING_UP` → `RUNNING` → `MARKET_CLOSED` → back
  to waiting, plus `ERROR_RETRY` reachable from anywhere) that keeps the
  dashboard/API available and auto-resumes at the next session with no
  manual restart. See [Flow, Strategy Logic & Diagrams](flow-and-examples.md#1-the-process-never-exits-because-the-market-is-closed).
- **Trading runtime and backtests are fully separate.** `APP_MODE` selects
  only `paper` or `live` at startup — there is no longer a "backtest"
  startup mode. Backtests are explicitly triggered from the dashboard as
  an on-demand job and run in an isolated per-run SQLite database and
  in-memory Redis, completely independent of the live market feed, the
  real PostgreSQL/Redis, and whether the paper/live runtime is active.
- **Paper and backtest execution are simulated**: they can consume real
  market/historical data but never submit real broker orders.
- **Live orders require independent safeguards**: selecting `APP_MODE=live`
  is not an authorization by itself. Real submission additionally
  requires valid Kite credentials, `APP_ENV=production`, and the trusted
  environment-only `ORDER_EXECUTION_ENABLED=true` gate. With that gate
  off, the live runtime is monitor-only.
- **The dashboard remains available when saved runtime settings are
  invalid**: the selected trading workflow is withheld and the error is
  exposed at `GET /api/runtime` (and the overall process state at
  `GET /api/status`) so an authorized operator can correct the settings
  and restart.
- **Idea clustering and duplicate-entry protection**: by default, only one
  entry per "idea" (same underlying + direction within a configurable
  clustering window, default 20 minutes) is allowed — the old unlimited-
  pyramiding behavior is still available behind an explicit flag, never
  the default. Repeated signal IDs are deduplicated independently of this.
- **The daily circuit breaker**: after a configurable number of
  consecutive *closed, losing* trades in one day (default 2), new entries
  are blocked or halved for the rest of the day, reset at the next trading
  day (and, by default, on any winning trade). See
  [Flow, Strategy Logic & Diagrams](flow-and-examples.md#the-daily-circuit-breaker).
- **Layered risk limits**: per-trade risk, per-idea risk budget, total
  open-risk cap, max exposure, max daily loss, max daily trades, and max
  open positions are all independently configurable (each defaults to "not
  enforced" except where a safe default makes sense — see
  [Setup & Configuration](setup.md)) and are additive, never a single
  point of failure.
- **Immediate order-entry pause**: the dashboard's Redis-backed control
  blocks new entries in paper and live runtimes (and in a running
  backtest). The execution engine rechecks the control immediately before
  submission; inability to read control state fails closed for entries.
  Protective and manual exits are never blocked by this.
- **Manual exits are explicit and auditable**: authorized dashboard
  operators can select one or more open paper/live position lots. Each
  submitted exit records `MANUAL_EXIT` and the targeted position ID.
  Backtest results are historical and cannot be manually exited.
- **No secrets in logs**: `JsonFormatter` redacts known credential field
  names even if they end up in `extra=` fields; `ConfigurationError`
  messages only ever mention variable names, never values.
  `PostgresClient` additionally redacts the full DSN (which embeds the
  password) from any connection-error text before it is logged or raised.
- **PostgreSQL/Redis health checks never block startup**: both clients are
  constructed lazily, so a temporarily unreachable database/cache never
  prevents the app from starting; `app.health()` reports the failing
  dependency until connectivity is restored.
- **Holiday calendar refresh**: the app fetches NSE CM/FO holidays at
  startup and stores them in PostgreSQL, falling back to the last
  persisted calendar plus a static fallback if NSE is unavailable. This
  still depends on NSE publishing a complete, parseable calendar; verify
  unexpected or ad hoc closures operationally.
- **Redis is never authoritative**: it's a fast-access cache layer only —
  PostgreSQL (and/or broker state) remains the durable source of truth,
  and every Redis failure is wrapped in a typed error, never silently
  swallowed or defaulted.
- **Every displayed/exported number is rounded consistently** (2 decimal
  places, `ROUND_HALF_UP`, applied only at the output boundary) — this is
  a display convention, not a computation shortcut; nothing about it
  changes what a signal or risk check would have decided.

**Equity swing is long-only and locally protected.** The configured equity
profile uses daily candles, CNC delivery, a 2x ATR stop (5% fallback when
there is insufficient ATR history), and a 2R target. SELL signals close an
existing long and never open short equity positions. The stop is not held
at the broker: overnight gaps can pass through it, and a stale/
disconnected feed, rejected exit, or market halt can delay or prevent the
application-submitted exit. Options positions use the intraday profile and
are scheduled for square-off at the configured IST cutoff.

## Known current gaps (honest, as of this writing)

These are real, current limitations — not hidden, not silently patched
over:

- **Conditional entry modes are backtest-only.** `risk_entry_mode` supports
  `next_open` (default, everywhere), plus `break_of_signal_bar_high` and
  `pullback_entry`, which only affect the **backtest engine** today — live
  and paper trading always fill at the next bar's open regardless of this
  setting. Extending the conditional-entry watcher to live/paper is future
  work.
- **The daily circuit breaker's `count_by="idea"` option currently behaves
  identically to `"trade"`.** Idea-level clustering exists as a primitive
  elsewhere in the risk engine, but wasn't retrofitted into the circuit
  breaker's own counting in this round.
- **No live, in-dashboard virtualized log viewer for backtests yet.**
  Every processed candle, trade event, and lifecycle event is logged
  (structured JSON) and exportable via the Run Log export sheet; a
  dedicated streaming/virtualized-scroll UI panel for tailing a running
  backtest's log in the browser hasn't been built.
- **Strategy validation and production approval remain open.** Both
  `v1_score` and `v2_setup` are implemented and test-covered, but neither's
  thresholds/parameters have been independently validated against real
  market performance, paper-traded for a meaningful stretch, and
  operationally reviewed. Implementation + passing tests is not the same
  claim as "ready to risk real capital." See
  [Strategy Specification](strategy-specification.md).
- **Broker capital/margin** (`AccountState.available_capital`/
  `used_capital`/`available_margin`) remain `None` — reconciliation
  compares orders and net positions, not broker capital or margin.
- **Option liquidity checks** (bid/ask spread, OI/volume thresholds) are
  not implemented anywhere (`models/tick.py`'s `bid`/`ask` fields exist but
  are unpopulated/unused). Validate liquidity yourself before trading
  anything beyond the most liquid NIFTY/BANKNIFTY contracts.
- **Live protective-exit dependency on the tick feed.** Live stop-loss/
  target monitoring depends on ticks reaching the application; if the Kite
  WebSocket disconnects or ticks go stale, the exit monitor cannot react
  until the feed recovers.
- **Reconciliation never generates a signal, sizes a position, or places a
  corrective order.** An unresolved broker/local mismatch is always left
  `REQUIRES_MANUAL_ACTION` — a human has to look at it.
- **The literal "28 trades / net +15,203 / profit factor 1.38" `v1_score`
  baseline, and the specific Nov 2/Nov 16 circuit-breaker fixtures, are the
  project owner's own prior manual analysis** — they are not numbers this
  repository's test suite reproduces (there's no historical market data
  for those exact dates/trades in this codebase). The underlying mechanisms
  (the scorer, the circuit breaker) are covered by synthetic, repeatable
  tests instead; reproducing the literal historical numbers would need the
  owner's own trade/candle data supplied and verified separately.
