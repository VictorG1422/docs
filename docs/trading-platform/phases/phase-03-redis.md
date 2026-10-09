# Phase 3 — Redis State & Cache Layer

Phase 3 adds the fast, non-durable runtime-state layer only: a pooled
Redis connection manager, a centralized key-naming scheme, a generic
state manager, and purpose-built stores for market/strategy/position
state, a distributed lock, a duplicate-event guard, and a generic cache.
It intentionally does **not** touch Kite Connect, the WebSocket, option
chains, the trading strategy, or order execution -- those remain
Phase 4+.

**Integration with the existing architecture:** Redis is wired in
exactly like PostgreSQL was in Phase 2 -- `Application` constructs a
`RedisClient` alongside the `PostgresClient`, registers a `"redis"`
health check next to `"postgresql"`, and disposes both on `stop()`.
Configuration reuses the same `Settings` (`pydantic-settings`) class and
`.env` file (no second configuration system); logging reuses the same
`configure_logging()`/`JsonFormatter`/`Event` machinery (no second
logging system); errors reuse the same `TradingSystemError` hierarchy
(`RedisUnavailableError(StorageError)`).

**What Phase 3 provides:**

- `trading_system/storage/redis/client.py::RedisClient` -- owns a pooled
  `redis.Connection`/`SSLConnection` (`redis.ConnectionPool`, not a new
  connection per operation), with configurable socket/connect timeouts,
  a `health_check()` (`PING`), and a graceful `close()`/`dispose()`.
  Constructing it never opens a real connection (lazy, like
  `PostgresClient`'s engine), so a briefly unavailable Redis never blocks
  application startup.
- `trading_system/storage/redis/keys.py::RedisKeyBuilder` -- the single,
  centralized place that builds Redis keys; nothing else in the codebase
  builds a raw key string. See [Redis key naming](../operations.md#redis-key-naming).
- `trading_system/storage/redis/serialization.py` -- explicit JSON
  `dumps`/`loads` helpers used by every store (never `pickle`).
  `datetime`/`date` serialize to ISO-8601 strings, `Decimal` serializes
  to a string (so prices/quantities/P&L/strikes never silently lose
  precision to a float), and `Enum` serializes to its `.value`.
- `trading_system/storage/redis/state.py::RedisStateManager` -- generic
  `set`/`get`/`delete`/`exists`/`expire`/`set_json`/`get_json`. Every
  Redis error (including corrupt JSON on read) is raised as
  `RedisUnavailableError` and logged via `Event.REDIS_ERROR`, never
  silently swallowed. All higher-level stores below build on this
  instead of talking to `redis-py` directly.
- `trading_system/storage/redis/market_state.py::MarketStateStore` --
  `set_latest_tick`/`get_latest_tick`/`delete_latest_tick`, keyed by
  instrument token, always TTL'd (default 60s) since a "latest tick" is
  inherently transient. **Storage interface only** -- nothing here
  subscribes to Kite or ingests a real tick; the future flow is
  `Kite WebSocket -> Tick Processor -> MarketStateStore -> Strategy`.
- `trading_system/storage/redis/strategy_state.py::StrategyStateStore` --
  `get_state`/`set_state`/`update_state`/`clear_state` for opaque,
  caller-defined per-strategy runtime state (last signal/entry/exit,
  trade count, cooldown, etc.), with **no TTL by default** -- this state
  must survive until the strategy explicitly clears it, not expire
  mid-session. Implements no actual strategy rules or indicators.
- `trading_system/storage/redis/position_state.py::PositionStateStore` --
  a fast-access cache (`get_position`/`set_position`/`delete_position`)
  for hot-path reads. **Redis is never the authoritative position
  store** -- PostgreSQL (`storage.db.models.Position`) and/or broker
  state remain the source of truth.
- `trading_system/storage/redis/lock.py::DistributedLock` -- see
  [Redis locks](../operations.md#redis-locks).
- `trading_system/storage/redis/dedup.py::DeduplicationStore` --
  `check_and_mark(event_id, ttl)`/`is_duplicate(event_id)`, both backed
  by an atomic `SET NX EX` (never a `GET`-then-`SET` race), for
  duplicate signal/event protection. TTL-bounded (24h default) so the
  dedup keyspace never grows unbounded.
- `trading_system/storage/redis/cache.py::CacheStore` -- a generic,
  explicit-purpose `set`/`get`/`delete` with a default TTL (5 minutes),
  intended for instrument lookups, option contracts, or other derived
  data -- nothing is cached automatically just because it can be.
- `config/settings.py::Settings` gains `redis_ssl`, `redis_socket_timeout`,
  and `redis_connect_timeout` alongside the existing `redis_host`/
  `redis_port`/`redis_password`/`redis_db`.
- `trading_system/exceptions/__init__.py::RedisUnavailableError` --
  distinguishes an unavailable Redis (cache/fast-state) from an
  unavailable PostgreSQL (durable store); which failure mode should
  fail-open vs. fail-closed is a Phase 4+ execution/risk decision, not
  decided here.
- `trading_system/utils/logger.py::Event` gains `REDIS_CONNECTED`,
  `REDIS_DISCONNECTED`, `REDIS_HEALTH_CHECK`, `REDIS_ERROR`,
  `REDIS_LOCK_ACQUIRED`, `REDIS_LOCK_FAILED`. Redis passwords are never
  logged (already covered by `JsonFormatter`'s `_SENSITIVE_KEYS`
  redaction), and no per-tick data is logged at `INFO` level.

**Design decisions / Phase 4+ TODOs:**

- **No Lua scripting dependency**: `DistributedLock.release_lock()` uses
  a `WATCH`/`MULTI`/`EXEC` transaction (not `EVAL`/`EVALSHA`) to check
  lock ownership before deleting, so it works identically against real
  Redis and against `fakeredis` in tests.
- **Testing**: unit tests (`tests/unit/test_redis_*.py`) run against
  `fakeredis.FakeRedis` (injected via `RedisClient(..., client=...)`), so
  they are hermetic and never touch a real Redis instance. The existing
  integration test (`tests/integration/test_storage_integration.py`)
  exercises `RedisClient.health_check()` against a real Redis, gated
  behind `RUN_INTEGRATION_TESTS=1` like the PostgreSQL integration test.
- **Nothing wired into the trading pipeline yet**: `TickProcessor`,
  `OrderManager`, and `RiskManager` do not call any of these stores
  yet -- that wiring (subscribing to real ticks, using `DistributedLock`
  around order submission, using `DeduplicationStore` for signal ids) is
  explicitly Phase 4+.

See the original prompt: [specs/03-phase3.md](../specs/03-phase3.md).
