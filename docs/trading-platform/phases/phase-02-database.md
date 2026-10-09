# Phase 2 — PostgreSQL Database Layer

Phase 2 adds the durable persistence layer only: SQLAlchemy ORM models,
Alembic migrations, a repository (data-access) layer, and a PostgreSQL
health check wired into the Phase 1 `Application`. It intentionally does
**not** touch Redis, the Kite API/WebSocket, order execution, or any
trading logic -- `OrderManager`/`PositionManager`/`RiskManager` do not
call the new repositories yet.

**What Phase 2 provides:**

- `trading_system/storage/db/base.py` -- `Base` (SQLAlchemy 2.x
  `DeclarativeBase`), a shared `utcnow()` helper, and `TimestampMixin`
  (`created_at`/`updated_at`, always tz-aware UTC).
- `trading_system/storage/db/models.py` -- ORM models for `Instrument`,
  `Signal`, `Order`, `Position`, and `Trade`, with indexes/constraints
  matching the spec (unique `instrument_token`, positive-quantity checks,
  FK relationships, native enum columns). This is a **separate** model
  layer from the Pydantic `trading_system.models` package used by the
  in-memory strategy/risk/execution pipeline -- business logic never
  depends on database column types, and the persistence layer never
  depends on in-memory pipeline concerns.
- `trading_system/storage/db/repositories.py` -- a generic
  `BaseRepository` plus `InstrumentRepository`, `SignalRepository`,
  `OrderRepository`, `PositionRepository`, `TradeRepository`, each
  operating on an injected `Session` (transaction boundaries stay with
  the caller via `PostgresClient.session()`).
- `trading_system/storage/postgres.py::PostgresClient` -- unchanged
  wrapper, hardened with a `connect_timeout` for PostgreSQL DSNs and DSN
  redaction so a connection failure can never leak the password into
  logs or raised exceptions.
- `trading_system/application.py::Application` now always constructs a
  `PostgresClient` (constructing it never opens a real connection) and
  registers a `"postgresql"` health check; `stop()` disposes the engine.
  A live database is **not** required for the application to start --
  `app.health()` simply reports `postgresql: false` if it is
  unreachable, matching Phase 1's "clearly report, don't crash" health
  philosophy.
- `migrations/versions/0001_initial_schema.py` -- creates `instruments`,
  `signals`, `orders`, `positions`, `trades` (see
  [PostgreSQL setup](../operations.md#postgresql-setup) and
  [Database migrations](../operations.md#database-migrations)).

**Design decisions / Phase 3+ TODOs:**

- **ID strategy**: every table uses a surrogate UUID primary key (`id`);
  broker-assigned identifiers (`Order.broker_order_id`) are always a
  separate, independently indexed column, never conflated with the
  internal id.
- **`OrderStatus` divergence (documented, intentional)**: the persisted
  `storage.db.models.OrderStatus` (`PENDING`/`OPEN`/`PARTIALLY_FILLED`/
  `FILLED`/`CANCELLED`/`REJECTED`/`FAILED`) is more granular than the
  in-memory `trading_system.models.order.OrderStatus`
  (`PENDING`/`OPEN`/`COMPLETE`/`REJECTED`/`CANCELLED`/`PARTIAL`) used by
  `OrderManager` today. Reconciling the two (a mapping function) is a
  Phase 3+ TODO, once `OrderManager` actually persists orders.
  Everywhere else, enums are reused as-is from `trading_system.models`
  (`InstrumentType`, `TransactionType`, `OrderType`, `ProductType`,
  `PositionStatus`, `SignalType`) to avoid duplicating identical value
  sets.
- **`metadata` columns**: SQLAlchemy declarative classes reserve the
  `metadata` attribute name, so the `Order`/`Signal` JSON metadata
  columns are mapped as `order_metadata`/`signal_metadata` in Python
  while still using the column name `metadata` in the database.
- **Tick data is still not persisted to PostgreSQL** -- unchanged from
  Phase 1; ticks remain a Redis/S3 concern.
- **No auto-DDL at startup**: nothing in `Application`/`PostgresClient`
  ever calls `Base.metadata.create_all()`/`drop_all()`; schema is owned
  exclusively by Alembic migrations.

See the original prompt: [specs/02-phase2.md](../specs/02-phase2.md).
