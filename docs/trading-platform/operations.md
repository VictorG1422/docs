# Operations

## Running in paper mode

```powershell
.\.venv\Scripts\Activate.ps1
python -m trading_system.main
```

What happens when you run this:

1. `get_settings()` loads and validates configuration from `.env`
   (falling back to safe defaults, `TRADING_MODE=paper` among them). A
   `ConfigurationError` here (e.g. an invalid `TRADING_MODE`) is caught
   before logging is configured, logged safely via `logging.basicConfig`,
   and returns exit code `1`.
2. `configure_logging()` installs structured JSON logging to stdout and
   a rotating file under `logs/`.
3. `SIGINT`/`SIGTERM` handlers are installed so the process can shut down
   cleanly instead of crashing.
4. `Application.start()` logs the loaded configuration and trading mode,
   then marks the application running (`"Application started
   successfully"`).
5. The application logs a health check result and then blocks, waiting
   for `Ctrl+C` or `SIGTERM`.
6. On shutdown (or any `TradingSystemError` during startup),
   `Application.stop()` always runs in a `finally` block and logs
   `"Application shutdown"`.

This intentionally does not connect to Kite, and does not subscribe to
any market data or place any order. As of Phase 4, step 5's health check
does attempt real PostgreSQL, Redis, and Kite connections (reporting
`postgresql: false`/`redis: false`/`kite: false` rather than crashing if
any is unreachable) -- see
[Safety Mechanisms & Roadmap](safety-and-limitations.md#what-remains-to-be-implemented)
for what later phases add on top of this foundation.

## Running with Docker

```powershell
docker build -t algo-trading .
docker run --rm --env-file .env algo-trading
```

The Dockerfile installs `requirements.txt`, then installs
the package itself (`pip install -e .`), and hardcodes
`ENV TRADING_MODE=paper` as a safe default that an operator must
deliberately override (e.g. `docker run -e TRADING_MODE=live ...`) to
enable live trading. The container's entrypoint is
`python -m trading_system.main`. PostgreSQL/Redis are expected to run as
separate services (e.g. via `docker-compose` or managed instances) --
point `POSTGRES_HOST`/`REDIS_HOST` at them via the env file.

## PostgreSQL setup

Any local PostgreSQL 13+ works. The quickest option is Docker:

```powershell
docker run --name algotrading-postgres -e POSTGRES_USER=postgres `
  -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=algotrading `
  -p 5432:5432 -d postgres:16
```

Then point the application at it via `.env` (defaults already match the
command above):

```
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=algotrading
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

Verify connectivity and apply the schema:

```powershell
python scripts/health_check.py   # optional: confirms PostgreSQL is reachable
alembic upgrade head             # creates instruments/signals/orders/positions/trades
```

`Settings.postgres_dsn` builds the connection string from these
variables (`postgresql+psycopg2://user:password@host:port/database`);
nothing needs to be configured separately for Alembic or the
application -- both read the same `Settings`.

## Database migrations

Schema is managed with Alembic (never create tables manually or via
`Base.metadata.create_all()`):

```powershell
alembic upgrade head           # apply all migrations
alembic revision --autogenerate -m "message"  # generate a migration from model changes
alembic downgrade -1           # roll back the last migration
```

`alembic.ini` and `migrations/env.py` read the database URL from
`Settings.postgres_dsn` so no separate DB URL configuration is needed.
`migrations/env.py` sets `target_metadata = Base.metadata` (imported from
`trading_system.storage.db.base`, which imports
`trading_system.storage.db.models` for its side effect of registering
every ORM class), so `--autogenerate` reflects future model changes.

The initial migration (`migrations/versions/0001_initial_schema.py`)
creates five tables -- `instruments`, `signals`, `orders`, `positions`,
`trades` -- with the same indexes, foreign keys, unique/check
constraints, and native PostgreSQL enum types as the ORM models in
`trading_system/storage/db/models.py`. Raw high-frequency ticks are
intentionally **not** persisted to PostgreSQL.

## Redis setup

Any local Redis 6+ works. The quickest option is Docker:

```powershell
docker run --name algotrading-redis -p 6379:6379 -d redis:7
```

Then point the application at it via `.env`:

```
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_SSL=false
REDIS_SOCKET_TIMEOUT=5
REDIS_CONNECT_TIMEOUT=5
```

`Settings` (`config/settings.py`) reads these the same way it reads the
PostgreSQL variables; nothing needs separate configuration wiring.

Verify connectivity:

```powershell
python scripts/health_check.py   # reports postgres/redis/broker health, exits 1 on any failure
```

or from a Python shell:

```python
from config.settings import get_settings
from trading_system.storage.redis import RedisClient

settings = get_settings()
client = RedisClient(
    host=settings.redis_host, port=settings.redis_port, password=settings.redis_password,
    db=settings.redis_db, ssl=settings.redis_ssl,
)
print(client.health_check())  # True if Redis is reachable
```

`Application.health()` also reports `redis: true/false` once the app is
running (see [Running in paper mode](#running-in-paper-mode)).

## Redis responsibilities

| | PostgreSQL | Redis |
| --- | --- | --- |
| Role | Durable source of truth | Fast, non-durable runtime state |
| Stores | instruments, orders, positions, trades, signals, historical records | latest market snapshots, strategy runtime state, a fast position-state cache, distributed locks, dedup, generic cache, cached option chains |
| Survives a restart | Yes, always | Only if TTL/eviction hasn't cleared it -- never relied upon |
| Managed by | Alembic migrations | `RedisKeyBuilder` + the Phase 3 stores (no schema/DDL) |

```
             ┌──────────────┐
             │ PostgreSQL   │
             │ Durable      │
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │ Trading App  │
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │ Redis        │
             │ Fast State   │
             └──────────────┘
```

Redis must never become the sole source of truth for durable business
state -- it exists purely so hot-path code (strategy/risk/order logic)
can avoid a PostgreSQL round trip for data that changes far more often
than it needs to be durably persisted.

## Redis key naming

Every Redis key is built by `trading_system.storage.redis.keys.RedisKeyBuilder`
-- no other module constructs a raw key string. Convention:

```
trading:{environment}:{kind}:{id}
```

| Kind | Example | Used by |
| --- | --- | --- |
| `market` | `trading:development:market:256265` | `MarketStateStore` (latest tick/OHLC/OI snapshot) |
| `tick` | `trading:development:tick:256265` | reserved for future raw-tick storage |
| `position` | `trading:development:position:256265` | `PositionStateStore` |
| `strategy` | `trading:development:strategy:placeholder` | `StrategyStateStore` |
| `lock` | `trading:development:lock:order-exec` | `DistributedLock` |
| `dedup` | `trading:development:dedup:<signal-id>` | `DeduplicationStore` |
| `cache` | `trading:development:cache:instrument:256265` | `CacheStore` |
| `option_chain` | `trading:development:option_chain:NIFTY:2024-12-26` | `OptionChainCacheStore` (Phase 5) |

`{environment}` is `Settings.app_env` (`development`/`paper`/`production`,
etc.), so different environments sharing one Redis instance can never
collide on the same key.

## Redis locks

`trading_system.storage.redis.lock.DistributedLock` exists to prevent
duplicate/concurrent execution of the same logical operation (e.g. two
processes both trying to act on the same signal):

```
Strategy generates signal -> acquire_lock(name, ttl) -> check state
    -> create order -> release_lock(name)
```

- **Acquisition** uses atomic `SET key value NX EX ttl` -- never a
  `GET`-then-`SET` race.
- **Release** uses a per-holder token checked inside a `WATCH`/`MULTI`
  transaction, so a lock can never be released by a holder that doesn't
  currently own it (e.g. after its TTL already expired and someone else
  acquired it).
- **TTL-bounded**: a crashed process can never hold the lock forever.

## Utility scripts

```powershell
python scripts/health_check.py          # verify Postgres/Redis/Kite connectivity, exits 1 on any failure
python scripts/download_instruments.py [EXCHANGE]  # sync the Kite instrument dump into PostgreSQL (default: NFO)
```

`health_check.py` is suitable for use as a container/orchestrator
readiness probe. `download_instruments.py` upserts instruments into the
`instruments` table via `InstrumentSyncService` -- safe to run
repeatedly, existing rows are updated in place rather than duplicated.

## Logging

`configure_logging()` (`utils/logger.py`) installs one consistent setup
for the whole application -- no module configures its own logging.
Records go to two places:

- **stdout**, as single-line JSON via `JsonFormatter` (ships directly to
  CloudWatch Logs or any JSON-aware log pipeline).
- **`logs/trading_system.log`**, the same JSON format, via a
  `RotatingFileHandler` (5 MB per file, 5 backups kept) so logs never
  grow unbounded. Generated log files are `.gitignore`d.

Example record:

```json
{"timestamp": "2026-08-11T09:15:00+00:00", "level": "INFO", "component": "trading_system.application", "event": "Trading mode: PAPER", "app_env": "development"}
```

Every record includes `timestamp`, `level`, `component` (logger name via
`logging.getLogger(__name__)`), and `event` (the log message), plus any
extra structured fields (e.g. `instrument`, `order_id`, `signal`).
Canonical event codes for major lifecycle/trading events live in
`utils/logger.py::Event` (`APPLICATION_STARTED`, `BROKER_CONNECTED`,
`ORDER_FILLED`, `RISK_LIMIT_TRIGGERED`, etc.).

**Sensitive values are never logged.** As defense in depth, `JsonFormatter`
also redacts any extra field whose key matches a known secret name
(`kite_api_key`, `kite_api_secret`, `kite_access_token`,
`postgres_password`, `redis_password`, `aws_access_key_id`,
`aws_secret_access_key`) to `***REDACTED***`, even if a future change
accidentally passes one through `extra=`.

Set `LOG_LEVEL=DEBUG` in `.env` for verbose logs (respected by both the
console and file handlers).
