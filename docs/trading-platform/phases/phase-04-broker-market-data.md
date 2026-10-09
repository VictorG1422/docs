# Phase 4 — Broker Integration & Market Data Foundation

Phase 4 wires the Kite Connect broker and the market-data foundation into
the existing Phase 1-3 architecture: typed authentication/error handling,
a Kite health check, instrument synchronization into PostgreSQL, and a
Reactive-free tick pipeline that writes the latest per-instrument snapshot
into the Phase 3 `MarketStateStore` (Redis). It intentionally does
**not** implement the trading strategy, signal generation, option-chain
selection, order placement, or risk management -- those remain Phase 5+.

**Integration with the existing architecture:** `Application` now also
constructs a `KiteBroker` (lazily -- like `PostgresClient`/`RedisClient`,
its constructor never makes a network call) and registers a `"kite"`
health check next to `"postgresql"`/`"redis"`. The tick pipeline
(`MarketState` + `TickProcessor`) is wired to a `MarketStateStore` built
from the existing Redis client/key-builder, and a `SubscriptionManager`
wraps `Broker.subscribe`/`unsubscribe` -- but nothing subscribes to any
instrument token yet; deciding *which* tokens to trade is a strategy
decision for a later phase.

**What Phase 4 provides:**

- `config/settings.py::Settings` gains `kite_request_timeout` (7s),
  `kite_ws_timeout` (30s), `kite_reconnect_enabled` (`true`), and
  `kite_max_reconnect_attempts` (5), all validated at startup (must be
  positive) via `_validate_kite_timeouts`.
- `trading_system/exceptions/__init__.py` gains a broker-specific
  exception hierarchy under `BrokerError`: `BrokerAuthenticationError`
  (→ `BrokerTokenExpiredError`), `BrokerAuthorizationError`,
  `BrokerConnectionError`, `BrokerTimeoutError`, `BrokerRateLimitError`,
  `InvalidBrokerRequestError`, `BrokerUnavailableError`, plus
  `InstrumentSyncError`. `broker/kite_client.py::_translate_kite_exception`
  maps every `kiteconnect.exceptions.*` error onto one of these -- the
  rest of the app never needs to know about `kiteconnect.exceptions`.
- `trading_system/broker/kite_client.py::KiteBroker` gains a
  `health_check()` (calls `profile()`, raising a typed error instead of
  crashing the app), `login_url()`/`generate_access_token()` for the
  normal Kite login flow (API key → login → request token → access
  token; **no browser automation**), and configurable request/WebSocket
  timeouts and reconnect settings passed through to `KiteWebSocketClient`.
- `trading_system/broker/websocket.py::KiteWebSocketClient` gains a
  `ConnectionState` (`DISCONNECTED`/`CONNECTED`/`RECONNECTING`), bounded
  reconnection (`reconnect_max_tries` from `KITE_MAX_RECONNECT_ATTEMPTS`,
  never an infinite tight loop), and automatic subscription restoration
  once reconnected.
- `trading_system/market_data/instrument_sync.py::InstrumentSyncService`
  -- `Kite -> validation -> InstrumentRepository -> PostgreSQL`. Matches
  existing rows by `instrument_token` (update in place) instead of
  duplicating them, so it is safe to run repeatedly; malformed records
  (bad token/symbol/lot size/tick size) are skipped and counted, not
  inserted. Uses the existing Phase 2 `PostgresClient.session()`/
  `InstrumentRepository` -- no second database layer.
- `trading_system/market_data/subscription_manager.py::SubscriptionManager`
  -- tracks the active instrument-token subscription set and makes
  subscribe/unsubscribe idempotent. Independent of any strategy; never
  hardcodes NIFTY/BANKNIFTY/expiry/strike.
- `trading_system/market_data/tick_processor.py::TickProcessor` now
  rejects obviously malformed ticks (non-positive instrument token,
  negative price) before they reach `MarketState` or Redis, and
  optionally forwards every valid tick into a `MarketStateStore` (a
  Redis outage here is logged and never breaks the tick pipeline, same
  isolation as a faulty listener). `MarketState.status()` classifies the
  latest tick per instrument as `FRESH`/`STALE`/`UNAVAILABLE` based on
  tick age -- independent of the Redis TTL, which only bounds how long
  stale data can accumulate.
- `trading_system/application.py::Application` constructs the
  `KiteBroker`, `MarketState`/`TickProcessor`/`MarketStateStore`, and
  `SubscriptionManager` described above; exposes `sync_instruments()`
  (delegates to `InstrumentSyncService`, safe to call repeatedly); and
  `stop()` now also calls `subscription_manager.unsubscribe_all()`.
- `trading_system/utils/logger.py::Event` gains `KITE_CONNECTED`,
  `KITE_AUTHENTICATED`, `KITE_AUTH_FAILED`, `KITE_TOKEN_EXPIRED`,
  `KITE_UNAVAILABLE`, `KITE_ERROR`, `KITE_WS_CONNECTED`,
  `KITE_WS_DISCONNECTED`, `KITE_WS_RECONNECTING`, `KITE_WS_RECONNECTED`,
  `KITE_SUBSCRIPTION_UPDATED`, `KITE_INSTRUMENT_SYNC_STARTED`/
  `_COMPLETED`/`_FAILED`. No per-tick data is logged at `INFO` level.
- `scripts/download_instruments.py` now persists the downloaded
  instrument dump into PostgreSQL via `InstrumentSyncService` (accepts
  an optional exchange argument, default `NFO`) instead of only
  reporting a count.

**Design decisions / Phase 5+ TODOs:**

- **Nothing subscribes to a real instrument token yet.** `SubscriptionManager`
  and the WebSocket infrastructure exist, but which NIFTY/BANKNIFTY
  contracts to stream is a strategy decision left to Phase 5+.
- **`sync_instruments()` is not called automatically at startup** -- it
  is exposed as an explicit operation (e.g. for a scheduled job or
  manual invocation) rather than run on every `Application.start()`.
- **Testing**: `tests/unit/test_kite_client.py` and
  `test_kite_websocket.py` never hit the real Kite API -- the
  `kiteconnect.KiteConnect`/`KiteTicker` instances are replaced with
  stubs/mocks after construction. `test_instrument_sync.py` uses an
  in-memory SQLite-backed `PostgresClient`. `test_tick_processor.py`
  uses `fakeredis` via the existing `redis_state`/`redis_keys` fixtures.
  `tests/integration/test_kite_market_data_integration.py` exercises
  both `Kite/mock broker -> Instrument Sync -> PostgreSQL` and
  `Kite/mock broker -> Tick Processor -> Redis -> Market State` against
  **real** PostgreSQL/Redis (never the real Kite API -- a fake `Broker`
  stands in), gated behind `RUN_INTEGRATION_TESTS=1` like the Phase 2/3
  storage integration test.
- **Post-Phase-4 hardening**: Kite credentials now come from AWS Secrets
  Manager rather than `.env` (see [AWS Secrets Manager](../setup.md#aws-secrets-manager)),
  and `KiteBroker` sets the access token on the underlying `KiteConnect`
  client eagerly in `__init__` -- a bug fix so `health_check()` (which
  never called `connect()` first) doesn't send an unauthenticated
  `profile()` request.

See the original prompt: [specs/04-phase4.md](../specs/04-phase4.md).
