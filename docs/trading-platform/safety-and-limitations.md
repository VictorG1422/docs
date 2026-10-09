# Safety Mechanisms & Roadmap

## Safety mechanisms

- **Paper trading is the default** (`TRADING_MODE=paper`, applied even
  when the variable is unset); switching to live trading requires an
  explicit environment variable plus valid Kite credentials, enforced by
  `pydantic` validators that raise `ConfigurationError` otherwise.
- **Unsupported trading modes fail fast**: any `TRADING_MODE` value other
  than `paper`/`live` (e.g. `production`) raises `ConfigurationError`
  before the application starts, instead of silently falling through.
- **No trading logic runs yet**: Phase 1's `Application` never connects
  to a broker or places an order; the previously-built `PlaceholderStrategy`
  always returns `HOLD`/`None`, and `RiskManager.validate_signal()`
  explicitly rejects `HOLD` signals, so no orders can be generated until
  a real strategy and Phase 2 wiring exist.
- **Duplicate-order protection**: `OrderManager` tracks submitted signal
  IDs and raises `DuplicateOrderError` on re-submission; `RiskManager`
  independently tracks seen signal IDs per trading day.
- **Risk limits with safe defaults**: `RiskLimits` defaults to a small
  `max_daily_loss` (5,000), `max_position_size` (1), and
  `max_trades_per_day` (10); daily counters reset automatically at
  midnight via `_roll_daily_state_if_needed`.
- **Emergency kill switch**: `RiskManager.emergency_shutdown()` blocks
  all further signals until explicitly reset.
- **Fail-fast startup**: a `ConfigurationError` while loading settings
  (invalid `TRADING_MODE`, missing live-trading credentials, etc.) is
  logged safely (no secrets) and returns exit code `1` before the
  application ever starts; `main()` always calls `Application.stop()` in
  a `finally` block, and `SIGINT`/`SIGTERM` trigger the same clean
  shutdown path instead of an abrupt process kill.
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

Phases 1-11 cover configuration, logging, the application lifecycle,
health checks (PostgreSQL + Redis + Kite + instrument data), the
PostgreSQL persistence schema/repository layer, the Redis state/cache/
lock layer, the Kite broker/instrument-sync/tick-pipeline foundation,
the instrument-discovery/option-chain foundation, the candle-
aggregation/historical-data/technical-indicator foundation, the
strategy/signal-generation framework (with a placeholder strategy, since
no real strategy specification exists yet), the risk-management/
position-sizing framework, Phase 9 order execution (gated, disabled by
default), Phase 10 broker reconciliation, and Phase 11
backtesting/paper trading. See the individual
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
- Deciding which NIFTY/BANK NIFTY instrument token(s) and timeframe(s) a
  `StrategyEngine` should actually watch, and constructing/`start()`ing
  one (or more) from `Application`/`main.py` -- `StrategyEngine` exists
  and is fully tested but is deliberately not wired into `Application`
  yet. The same is true for `RiskEngine` (Phase 8).
- Wiring `TradingEngine` (or an equivalent) into `main.py`/`Application`
  to actually run the `StrategyEngine -> RiskEngine -> OrderExecutionEngine
  -> Broker` pipeline end-to-end.
- Wiring the legacy `OrderManager`/`PositionManager` (the Phase 1
  placeholder pipeline) to actually persist via the repository layer and
  the Phase 3 Redis stores -- the schema/repositories/stores exist, but
  nothing calls them yet.
- Using `DistributedLock` around real order submission beyond strategy/
  risk evaluation scope -- the primitive exists but a broader
  order-placement-scoped use is not yet wired.
- Calling `Application.sync_instruments()` on a schedule (or at startup)
  so `UnderlyingDiscoveryService`/`ExpiryService`/`OptionChainService`
  lookups always reflect the latest Kite instrument dump.
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
- **Phase 11 backtesting/paper-trading boundary** (see
  [Phase 11](phases/phase-11-backtesting-paper.md) for the full write-up):
  there is still no exit/position-closing pipeline anywhere in Phases
  7-9 (`RiskEngine` rejects any `EXIT` signal as "not risk-manageable")
  -- a backtested/paper position can only ever be opened, never closed by
  the strategy itself. No stop-loss/target *monitoring* against ongoing
  price movement exists either (Phase 8 only calculates and records a
  stop-loss/target value on the `TradeIntent`; nothing watches
  live/historical price action against it and emits an exit). Adding a
  real exit pipeline is squarely a Phase 12 concern -- Phase 11
  deliberately does not invent one. `RiskEngine._check_trading_window()`'s
  real-wall-clock IST check and Redis-TTL-based cooldown
  (`RiskPolicy.cooldown_seconds`) are not clock-injected for backtesting
  -- leave them unset for an accurate backtest. No background
  daemon/process manager exists for paper sessions yet
  (`scripts/paper_trade.py` runs in the foreground only).
