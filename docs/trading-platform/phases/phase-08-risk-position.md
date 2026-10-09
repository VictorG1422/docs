# Phase 8 — Risk & Position Management

Phase 8 builds the risk-management/position-sizing layer that sits
between Phase 7's `Signal` and Phase 9 order
execution. **It never places an order, never calls a Kite order-placement
API, and never generates a new BUY/SELL signal** -- it only approves or
rejects a signal Phase 7 already produced, and, if approved, sizes it
into a `TradeIntent` for Phase 9 to consume later.

```
Signal (Phase 7, status="RISK_CHECK_PENDING")
      │
      ▼
RiskEngine.evaluate_signal()
      │  DistributedLock (per-instrument, narrow, bounded)
      ▼
signal validation → market-data validation (staleness) → trading-window
validation → existing-position validation → duplicate-signal validation
→ cooldown validation → account-state load → daily-loss validation →
daily-trade validation → max-open-positions validation → exposure-room
validation → instrument/lot-size metadata → PositionSizer → quantity
validation → configured stop-loss/target/risk-reward calculation
      │
      ▼
RiskDecision (approved/rejected, every check outcome recorded)
      │
      ▼
TradeIntent  --> PostgreSQL (risk_decisions, trade_intents)
             --> Redis (PositionStateStore: instrument → PENDING)
```

- `trading_system/risk/policy.py::RiskPolicy` -- every limit
  (`max_risk_per_trade`, `max_daily_loss`, `max_open_positions`,
  `max_daily_trades`, `max_exposure`, `max_quantity`,
  `max_premium_exposure`, `min_risk_reward_ratio`, `cooldown_seconds`,
  `trading_window_start`/`end`, `data_staleness_limit_seconds`) defaults
  to `None` ("not enforced") rather than an invented rupee/percentage
  value -- no strategy specification exists yet to justify one (see
  [Phase 7](phase-07-strategy-signal.md) and
  [the strategy specification](../strategy-specification.md), still a
  template). `RiskPolicy.from_settings()` builds one from the new
  `RISK_*` environment variables on `config/settings.py::Settings` (the
  same, single configuration system used everywhere else -- no second
  config mechanism). `version` (`RISK_POLICY_VERSION`, default `"1"`) is
  recorded on every `RiskDecision`/`TradeIntent` so a later audit can tell
  exactly which risk configuration approved/rejected a given trade.
- `trading_system/risk/risk_decision.py` -- `RiskCheckStatus`
  (`PASSED`/`FAILED`/`SKIPPED` -- a `SKIPPED` check, e.g. an unconfigured
  limit or the not-yet-defined stop-loss, is never treated as an
  approval by itself), `RejectionReason` (machine-readable: e.g.
  `STALE_MARKET_DATA`, `MAX_DAILY_LOSS_EXCEEDED`, `MAX_EXPOSURE`,
  `DUPLICATE_POSITION`, `INSUFFICIENT_CAPITAL`, `RISK_LOCK_UNAVAILABLE`,
  `RISK_SYSTEM_UNAVAILABLE`), `RiskCheckOutcome` (one pipeline step,
  always recorded), and `RiskDecision` (the full, auditable result --
  every rejected/approved decision records every check that ran, not just
  the first failure).
- `trading_system/risk/account_state.py::AccountState`/
  `AccountStateProvider` -- `available_capital`/`used_capital`/
  `available_margin` are **always `None`**: real broker-account
  reconciliation is explicitly Phase 10's job, so Phase 8 never fabricates
  them. `PostgresAccountStateProvider` derives everything else from
  durable PostgreSQL records: `current_exposure`/`unrealized_pnl_today`
  from open `Position` rows, `realized_pnl_today` from `Trade` rows whose
  `exit_time` falls on the current day, and `daily_trade_count` from
  `RISK_APPROVED` `TradeIntent` rows created that day. Correct today even
  though nothing populates `positions`/`trades` until Phase 9 executes an
  order -- forward-compatible, not fabricated.
- `trading_system/risk/position_sizer.py::PositionSizer` -- `Signal +
  RiskPolicy + AccountState + entry price (+ optional stop-loss) ->
  quantity`. Every calculation uses `Decimal`, never `float`. Quantity is
  always rounded **down** to the nearest whole lot (`instrument.lot_size`,
  Phase 5 metadata -- never a hardcoded NIFTY/BANK NIFTY lot size); a
  result below one lot is reported as `INSUFFICIENT_CAPITAL`, never
  silently clamped to a nonzero quantity. Risk-based sizing
  (`max_risk_per_trade / |entry - stop_loss|`) activates automatically
  once `max_risk_per_trade` is configured, using the stop-loss computed
  below; otherwise sizing falls back to
  `max_quantity`/available-capital/exposure-room limits, or exactly one
  lot if nothing is configured at all.
- **Stop-loss & target (`trading_system/risk/stop_loss.py`)** -- a
  production-standard, risk-engine-owned protective overlay, independent
  of the (still unwritten) entry strategy. `StopLossCalculator` defaults
  to a volatility-adaptive **ATR stop** (`distance = ATR(period) *
  multiplier`, Wilder's 14-period ATR by default, 1.5x multiplier) --
  the standard approach in production systems because it automatically
  widens in volatile markets and tightens in quiet ones, unlike a fixed
  distance. When there isn't yet enough candle history for a reliable
  ATR (e.g. right after startup), it **automatically and transparently
  falls back** to a percentage-based stop (10% of entry price by
  default) -- a missing *historical* data point never blocks a trade,
  and every fallback is recorded in the check detail for audit. A
  `POINTS` (fixed rupee distance) method is also supported. Every
  computed stop is clamped to a sane range: never <= 0 (floored at
  ₹0.05) and never more than 90% of the entry price/premium.
  `TargetCalculator` derives the target purely from a configured
  reward:risk ratio (`RiskPolicy.target_reward_ratio`, default 1.5x)
  applied to the realized stop-loss distance -- never an arbitrary price
  level. A final `risk_reward` check re-derives the *actual* achievable
  ratio from the (possibly clamped) stop-loss/target and rejects
  (`MIN_RISK_REWARD_NOT_MET`) if it falls short of
  `RiskPolicy.min_risk_reward_ratio` -- `RiskPolicy.__post_init__`
  additionally refuses to construct a policy where
  `target_reward_ratio < min_risk_reward_ratio` at configuration time.
  **Fully disable-able**: set `stop_loss_method`/`target_reward_ratio`
  to `None` (`RISK_STOP_LOSS_METHOD=`/`RISK_TARGET_REWARD_RATIO=` empty)
  to skip both entirely (`SKIPPED` checks, `stop_loss=None`/`target=None`
  on the `TradeIntent`, `metadata["stop_loss_configured"] = False`) --
  every parameter remains fully configurable/overridable via environment
  variables, this is not a second, hidden configuration mechanism.
- `trading_system/risk/position_state.py::PositionLifecycleStatus`/
  `RiskPositionStateStore` -- lifecycle semantics (`NONE`/`PENDING`/
  `OPEN`/`CLOSING`/`CLOSED`) layered on top of the existing Phase 3
  `PositionStateStore` (Redis) rather than a second position store. A
  risk-approved trade intent moves an instrument to `PENDING` --
  **never** `OPEN`: an approved intent is not yet an executed position
  (Phase 9's job). Also used as the "existing position" / duplicate-
  position gate: a new signal for an instrument that isn't `NONE` is
  rejected (`DUPLICATE_POSITION`).
- `trading_system/risk/risk_engine.py::RiskEngine` -- the orchestrator.
  Acquires a narrow, per-instrument `DistributedLock`
  (`risk_eval:{instrument_token}`) before evaluating (an already-held
  lock is rejected as `RISK_LOCK_UNAVAILABLE`, never queued/blocked);
  checks market-data freshness via `MarketDataService`/`MarketDataStatus`
  (+ an optional configurable staleness limit tighter than Phase 6's
  default); checks the trading window (IST, via
  `utils/time_utils.py::now_ist`/`is_trading_day` -- never a hardcoded
  timezone), which also enforces two **NIFTY/BANKNIFTY F&O-specific,
  opt-in** session-edge guards -- `RiskPolicy.avoid_first_minutes_of_session`/
  `avoid_last_minutes_of_session` (recommended: 15 minutes each,
  `RISK_AVOID_FIRST_MINUTES_OF_SESSION`/`RISK_AVOID_LAST_MINUTES_OF_SESSION`)
  reject trade intents near the 09:15/15:30 IST session open/close, where
  opening-range whipsaws and closing/pin-risk volatility are most
  common -- unset by default (unlike the stop-loss defaults) since they
  depend on wall-clock/session time, which must never silently change
  test/deployment behavior; checks for a duplicate signal ID via the existing Phase 3
  `DeduplicationStore` (own `risk_signal:{signal_id}` namespace, distinct
  from Phase 7's candle-dedup key); checks a per-strategy+instrument
  cooldown via the new `RiskRuntimeStateStore` (Redis, optional -- skipped
  if not configured/wired); loads `AccountState` and runs daily-loss/
  daily-trade/max-open-position/exposure-room checks (each `SKIPPED` if
  its policy limit is unset); loads Phase 5 instrument metadata (lot
  size, underlying, expiry, strike) via `InstrumentRepository`; runs
  `PositionSizer`; and finally builds a `TradeIntent`. **Fail-safe by
  construction**: any `StorageError`/`RedisUnavailableError` anywhere in
  the pipeline (including a persistence failure at the very end) is
  caught and converted into a `RISK_REJECTED` intent with
  `RISK_SYSTEM_UNAVAILABLE` and `metadata["persisted"] = False` -- a
  critical infrastructure failure can never become a silent approval, and
  the caller can always tell whether a decision was actually durably
  recorded.
- `trading_system/models/trade_intent.py::TradeIntent`/
  `TradeIntentStatus` -- Phase 8's sole output. `status` is one of
  `RISK_APPROVED`/`RISK_REJECTED`/`PENDING_EXECUTION` (Phase 8 never
  writes the latter -- reserved for Phase 9). Never `EXECUTED`/`FILLED`/
  `COMPLETED`. Fields mirror the check-pipeline output: `quantity`,
  `entry_price_reference`, `risk_amount`, `maximum_loss`,
  `capital_required`, configured `stop_loss`/`target`, `risk_checks`
  (every `RiskCheckOutcome`, serialized), and
  `risk_policy_version`.
- **Persistence (PostgreSQL, `migrations/versions/0006_risk_trade_intents.py`)**:
  new `risk_decisions` (every evaluation, approved or rejected --
  immutable/append-only, like `signals`) and `trade_intents` tables (FKs
  to `signals`/`risk_decisions`/`instruments`). A signal can in principle
  be re-evaluated (e.g. after a transient failure), producing a new row
  rather than mutating a prior decision, so the full audit history is
  always available. `RiskDecisionRepository`/`TradeIntentRepository`
  (`storage/db/repositories.py`) follow the existing `BaseRepository`
  pattern; `TradeRepository.list_by_exit_date()` and
  `TradeIntentRepository.count_approved_since()` back the
  `AccountStateProvider` queries above.
- **Redis usage (`storage/redis/risk_state.py::RiskRuntimeStateStore`)**:
  purely a fast, **non-authoritative** accelerator/cooldown store (atomic
  `INCR`+`EXPIRE` daily counters, `SET EX` cooldown markers) -- PostgreSQL
  (`AccountStateProvider`) remains authoritative for daily trade
  counts/P&L. A Redis failure here is never treated as "no risk state
  exists"; callers fall back to the PostgreSQL figures.
- **Concurrency**: the per-instrument `DistributedLock` above is the
  primary guard; the `DeduplicationStore` check additionally prevents the
  same signal ID from ever being risk-evaluated twice, even across
  process restarts.
- **Testing**: `tests/unit/test_risk_policy.py`, `test_position_sizer.py`
  (risk-based/capital-based sizing, lot-size flooring, zero-quantity
  rejection, exposure-room pre-check, Decimal precision), `test_account_state.py`
  (exposure/P&L/daily-trade-count aggregation against an in-memory SQLite
  PostgreSQL), `test_risk_state_store.py` (atomic counters/cooldowns
  against `fakeredis`), `test_stop_loss.py` (ATR/percentage/points stop-loss,
  automatic ATR-insufficient-history fallback, clamping bounds, BUY/SELL
  direction, reward:risk target derivation, disabled-config paths), and
  `test_risk_engine.py` (every check in the
  pipeline: valid approval, invalid/HOLD signal, missing/stale market
  data, invalid price, max open positions/exposure/daily-loss/daily-
  trades, insufficient-capital zero-quantity, duplicate signal, duplicate
  position, missing instrument metadata, trading window, session-edge
  (opening/closing) guards, lock
  unavailable, cooldown, stop-loss/target population, risk-based sizing
  activation, minimum-risk-reward rejection). `tests/integration/test_risk_engine_integration.py`
  exercises `Signal -> RiskEngine -> RiskDecision/TradeIntent ->
  PostgreSQL + Redis` end-to-end against real local Postgres/Redis
  (gated by `RUN_INTEGRATION_TESTS=1`, same pattern as the Phase 7
  integration test).
- **Remaining limitations**: broker capital/margin are always
  `None` until Phase 10 reconciliation; the ATR stop-loss is computed
  from the option contract's *own* price candles (appropriate for
  premium-based volatility) but a strategy specification may eventually
  want a different basis (e.g. the underlying's ATR) -- revisit once
  the strategy specification is completed; exposure only
  counts durably-open PostgreSQL positions plus the single trade being
  evaluated (a pending-but-unexecuted `TradeIntent` is deliberately not
  counted as exposure) -- multiple concurrently-approved intents across
  different instruments could still exceed `max_exposure` in aggregate
  before fills; `RiskEngine`/`RiskPolicy`/`PositionSizer`/`AccountStateProvider`
  are not yet wired into `Application`/`engine.py` (same deliberate
  choice Phase 7 made for `StrategyEngine` -- no concrete strategy/signal
  source is running yet).

See the original prompt: [specs/08-phase8.md](../specs/08-phase8.md).
