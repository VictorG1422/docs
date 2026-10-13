# Safety Mechanisms & Roadmap

## Safety mechanisms

- **One configured workflow runs per process**: PostgreSQL
  `APPLICATION_MODE` selects `backtest`, `paper` (the default), or `live`.
  Unsupported values are rejected instead of silently falling through.
- **Paper and backtest execution are simulated**: they can consume real
  market/historical data but never submit real broker orders.
- **Live orders require independent safeguards**: the DB-selected
  `APPLICATION_MODE=live` is not an authorization by itself. Real
  submission additionally requires valid Kite credentials,
  `APP_ENV=production`, and the trusted environment-only
  `ORDER_EXECUTION_ENABLED=true` gate. With that gate off, the live
  runtime is monitor-only.
- **The dashboard remains available when saved runtime settings are
  invalid**: the selected trading workflow is withheld and the error is
  exposed at `GET /api/runtime` so an authorized operator can correct the
  settings and restart.
- **Duplicate-order protection**: `OrderManager` tracks submitted signal
  IDs and raises `DuplicateOrderError` on re-submission; `RiskManager`
  independently tracks seen signal IDs per trading day.
- **Risk limits with safe defaults**: `RiskLimits` defaults to a small
  `max_daily_loss` (5,000), `max_position_size` (1), and
  `max_trades_per_day` (10); daily counters reset automatically at
  midnight via `_roll_daily_state_if_needed`.
- **Emergency kill switch**: `RiskManager.emergency_shutdown()` blocks
  all further signals until explicitly reset.
- **Fail-safe startup**: invalid persisted runtime configuration does not
  start a trading workflow. The unified launcher still serves the
  dashboard where possible, and Uvicorn owns graceful shutdown of the
  shared application lifecycle.
- **No secrets in logs**: `JsonFormatter` redacts known credential field
  names even if they end up in `extra=` fields; `ConfigurationError`
  messages only ever mention variable names, never values.
  `PostgresClient` additionally redacts the full DSN (which embeds the
  password) from any connection-error text before it is logged or raised.
- **PostgreSQL health check never blocks startup**: `Application`
  constructs a `PostgresClient` lazily (no connection attempt) so a
  temporarily unreachable database never prevents the app from starting;
  `app.health()` reports `postgresql: false` until connectivity is
  restored. The same is true for `RedisClient`/`redis`.
- **Redis is never authoritative**: `PositionStateStore`/`MarketStateStore`/
  `StrategyStateStore` are fast-access caches only -- PostgreSQL (and/or
  broker state) remains the durable source of truth.
- **Redis errors are never silently swallowed**: every Redis operation
  wraps failures in `RedisUnavailableError` (logged via `Event.REDIS_ERROR`)
  instead of returning a default value or crashing the whole process.

## What remains to be implemented

Phases 1-11 cover configuration, logging, the unified application
lifecycle, health checks, PostgreSQL and Redis persistence/state,
Kite integration, instrument synchronization, market data, strategy and
risk pipelines, gated order execution, reconciliation, backtesting, and
paper trading. The configurable generic `technical_scoring` strategy is
not a claim of a strategy validated or approved for production. See the individual
[phase write-ups](phases/phase-01-foundation.md) for full detail. Key
open items:

- **The actual trading strategy itself.** `NoOpStrategy` never returns a
  decision -- a real `BaseStrategy` implementation (entry/exit rules,
  indicator thresholds, option-selection rules) requires an actual
  strategy specification, which does not exist anywhere in this project
  (see [Phase 7](phases/phase-07-strategy-signal.md)). Complete
  [the strategy specification](strategy-specification.md) (added in the
  Phase 7A follow-up) to unblock this. (Phase 8's stop-loss/target are
  already implemented as a risk-engine-owned overlay -- see
  [Phase 8](phases/phase-08-risk-position.md) -- independent of whatever
  entry/exit rules this specification eventually defines.)
- Wiring the legacy `OrderManager`/`PositionManager` (the Phase 1
  placeholder pipeline) to actually persist via the repository layer and
  the Phase 3 Redis stores -- the schema/repositories/stores exist, but
  nothing calls them yet.
- Using `DistributedLock` around real order submission beyond strategy/
  risk evaluation scope -- the primitive exists but a broader
  order-placement-scoped use is not yet wired.
- Deciding which interval(s)/instruments to back-fill via
  `MarketDataService.get_historical_data()` (Phase 6) at startup so
  indicators have a warm-up window before the first live candle
  completes -- the retrieval path exists, but nothing calls it yet.
- End-of-day/session finalization for the last "forming" candle of a
  trading session (Phase 6's `CandleAggregator` only completes a candle
  when the *next* bucket's first tick arrives).
- **Phase 10 broker reconciliation boundary**: real broker capital/margin --
  `AccountState.available_capital`/`used_capital`/`available_margin` are
  always `None` (Phase 8/10 never fabricate them; Phase 10 reconciles
  orders/positions, not margin/capital). `Position`/`OrderManager`
  in-memory objects (the Phase 1 placeholder pipeline) are still
  separate from the persisted `storage.db.models.Position`/`Order` rows
  Phase 10 reconciles -- wiring the placeholder pipeline itself into
  `TradingEngine` remains open.
- NSE holiday calendar sourced from PostgreSQL/a periodic sync job
  instead of the static fallback list in `utils/time_utils.py`.
- Option-chain/tick/candle/signal snapshotting to S3.
- Additional risk checks once the strategy specification exists: margin,
  per-instrument exposure limits beyond what `RiskPolicy` already
  supports, option liquidity/bid-ask-spread checks (no such market data
  is captured anywhere yet -- see `models/tick.py`'s optional `bid`/`ask`
  fields, currently unpopulated).
- **Phase 11+ boundary**: Phase 10 reconciles existing broker/PostgreSQL/
  Redis state only -- it never generates a trading signal, never sizes a
  position, and never places a corrective order (an unresolved mismatch
  is always left `REQUIRES_MANUAL_ACTION`). Automatically acting on an
  unresolved reconciliation mismatch remains explicitly out of scope.
- **Protective-exit data dependency** (see
  [Phase 11](phases/phase-11-backtesting-paper.md)): backtests and paper
  sessions evaluate stop-loss/target against the traded instrument's own
  candles. Live protective exits depend on ticks reaching the application;
  if the Kite WebSocket disconnects or ticks become stale, the local
  tick-triggered exit monitor cannot react until the feed recovers. Orders
  are still gated by `ORDER_EXECUTION_ENABLED`, live mode, and production
  environment checks. `RiskEngine._check_trading_window()`'s
  real-wall-clock IST check and Redis-TTL-based cooldown
  (`RiskPolicy.cooldown_seconds`) are not clock-injected for backtesting.
  Paper sessions run as part of the unified `python -m trading_system.main`
  process and are configured through PostgreSQL dashboard settings.
