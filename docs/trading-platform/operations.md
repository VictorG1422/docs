# Operations

## Running the algorithm

```powershell
.\.venv\Scripts\Activate.ps1
python -m trading_system.main
```

`python -m trading_system.main` starts only the configured algorithm. It
loads secrets and validated PostgreSQL settings, synchronizes required
instruments, then runs backtest, paper, or live mode. It does not start an
HTTP server. Runtime values are managed in PostgreSQL; there are no mode,
strategy, instrument, or date command-line arguments.

At startup, the algorithm fetches the NSE trading-holiday calendar for CM
and FO from NSE and upserts it into PostgreSQL. If NSE is unreachable or
returns invalid data, it uses the last persisted calendar plus the static
fallback; a holiday-service failure alone does not stop trading startup.
Migration `0017` creates the holiday table. NSE calendar maintenance
applies to the algorithm process; a dashboard-only process does not sync it.
The dashboard reads cached rows through `GET /api/holidays/today` and shows
CM/FO descriptions in a banner when today is a holiday; it does not request
the NSE API itself. The **Market Holidays** page (`/holidays`) lists all
cached dates, segments, descriptions, and sync timestamps from
`GET /api/holidays`.

## Running the dashboard separately

```powershell
.\.venv\Scripts\Activate.ps1
python -m dashboard.main
```

The dashboard serves its UI and API at `http://127.0.0.1:8000/` by default.
Run it in its own terminal or container; it reads the same PostgreSQL and
Redis configuration as the algorithm and can run while the algorithm is
stopped. Dashboard setting changes are saved to PostgreSQL and take effect
when the algorithm next starts. `GET /api/runtime` reports that the runtime
is managed separately; use Backtests, Paper Sessions, logs, and health
checks for persisted status. The detached dashboard does not submit orders;
manual position exits are unavailable there.

`scripts/run_dashboard.py` is a compatibility shim for
`python -m dashboard.main`. An invalid database runtime configuration stops
the algorithm process while leaving the separately running dashboard
available to correct it.

If saving an options watchlist returns HTTP 422 with a missing CE/PE
contracts message, synchronize NSE and NFO instrument data with
`python scripts/download_instruments.py` and confirm both option types
exist at a common eligible expiry. A rejected save leaves the previous
setting unchanged.

Paper-mode restarts preserve completed session records under
`archived:<session UUID>` and create a fresh record for the configured
strategy/instrument/timeframe key. A record still marked `RUNNING` blocks
another session; do not delete historical rows to resolve a restart error.

WebSocket startup is visible at INFO level: `KITE_WS_CONNECT_ATTEMPT`,
`KITE_WS_CONNECTED`, `KITE_WS_SUBSCRIBE_ATTEMPT`, then
`KITE_SUBSCRIPTION_UPDATED`. The subscription update confirms the client
sent the commands, not a broker acknowledgement; a received tick verifies
delivery. `KITE_CONNECTED` alone confirms REST authentication only. A
paper-session startup failure occurs before WebSocket subscription, so
DEBUG mode does not produce a connection attempt in that case.
Kite Connect 5.x pins Autobahn 19.11.2; keep Twisted below 25.0 for
compatibility. Twisted 26.4.0 produced repeated WebSocket 1006 closures,
while 24.11.0 connected successfully in the same environment. Log
timestamps are shown in IST (+05:30). Ctrl+C invokes application shutdown,
unsubscribes tokens, dispatches the close call on Twisted's reactor thread,
and waits briefly for the close callback before aborting the local transport
if the peer does not complete the handshake.

## Trading profiles

The options watchlist uses the intraday profile: BUY signals select a CE,
SELL signals select a PE, and open option positions are squared off at
`OPTIONS_INTRADAY_SQUARE_OFF_TIME` (default `15:15` IST). The equity
watchlist uses a long-only swing profile: daily candles, CNC delivery,
ATR stop at `2.0x` with a `5%` fallback when ATR history is insufficient,
and a target at `2R`. An equity SELL closes an existing long; it never
opens a short. Swing positions are retained across paper-process restarts.

The profile values are editable under Trading & risk settings and are
stored in PostgreSQL. Run the migrations before starting this version;
`0015` adds durable paper-session portfolio state and `0016` seeds these
profile defaults without overwriting existing settings. A local stop is
not a broker-held stop: gaps, stale/disconnected market data, rejected
orders, and market halts can result in a worse exit or an unclosed
position. Validate the strategy and operating procedures before any live
use; tests do not constitute production approval.

## Daily pivot context

`technical_scoring` calculates classic pivots once from the previous
completed 1440-minute session candle. It records `P`, `BC`/`TC`, `R1`-`R3`,
`S1`-`S3`, CPR bounds/width, and the source session date in the signal
snapshot. Price above/below the CPR contributes one bullish/bearish
`pivot_context` score (default weight 8); crossing beyond R1/S1 increases
that feature's strength. Displayed pivot/CPR levels are rounded to two
decimal places; calculations retain full Decimal precision. Price inside
the CPR is neutral. R2/R3/S2/S3 are
context only, not automatic targets or exits. Set
`SCORING_WEIGHT_PIVOT_CONTEXT=0` in dashboard score settings to disable its
score contribution. If prior daily history is unavailable or invalid, the
feature is omitted and existing scoring continues. Backtests use only
daily candles from earlier IST session dates; the same-day daily bar is
never used to create that day's pivot levels.

Technical scoring also requires both entry gates: price must be above both
EMA(200) and session VWAP for bullish bias, or below both for bearish;
mixed/equal alignment blocks the signal. Positive-strength OI buildup or
volume spike is required as flow confirmation. Every evaluation is logged;
near-miss scores from 40 through 64 or -40 through -64 persist component,
gate, and pivot details. Migration `0018` creates diagnostics storage;
`0019` updates legacy default rows, and `0020` persists the requested score
weights with an audit entry while leaving thresholds and other settings
unchanged.

In OPTIONS mode, the gates use a synchronized, unexpired FUT contract,
because index candles do not provide OI and may have no usable volume. Its
close, EMA/VWAP, OI, and volume supply only macro/flow confirmation; the
index remains the pattern-scoring and signal source. Live and paper entries
wait for a matching futures candle, and missing or misaligned reference data
fails closed. Sync NFO futures before starting the OPTIONS runtime or a
backtest; the backtest fails validation if no contract covers the full run.

## Running with Docker

```powershell
docker build -t algo-trading .
docker run --rm --env-file .env algo-trading
```

The Dockerfile installs `requirements.txt`, then installs
the package itself (`pip install -e .`). `APPLICATION_MODE` defaults to
paper in PostgreSQL; the mode is changed through the protected dashboard.
Live order submission additionally requires the independently trusted
`ORDER_EXECUTION_ENABLED=true`, `APP_ENV=production`, and valid Kite
credentials. The default container command runs only
`python -m trading_system.main`. Run the dashboard in a separate container
with `docker run --rm --env-file .env -e DASHBOARD_HOST=0.0.0.0 -p 8000:8000
algo-trading python -m dashboard.main`. PostgreSQL/Redis are expected to
run as separate services (e.g. via `docker-compose` or managed instances) --
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
alembic upgrade head             # apply the complete database schema
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

Migration `0012` updates the instrument lot-size check to allow zero only
for `INDICES` reference rows; tradable instruments still require a positive
lot size. Migration `0011` adds dashboard score, settings, and audit tables.
Migration `0015` adds paper-session runtime state; `0016` idempotently seeds
the options/equity profile settings while preserving existing values; `0017`
creates the NSE CM/FO holiday cache.

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
running (see [Running the algorithm](#running-the-algorithm)).

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
python scripts/health_check.py                 # verify Postgres/Redis/Kite connectivity
python scripts/download_instruments.py          # sync both NFO and NSE
python scripts/download_instruments.py NSE      # sync one exchange
python scripts/download_instruments.py NSE NFO  # sync selected exchanges
```

`health_check.py` is suitable for use as a container/orchestrator
readiness probe. `download_instruments.py` upserts instruments into the
`instruments` table via `InstrumentSyncService` -- safe to run
repeatedly, existing rows are updated in place rather than duplicated.
NSE sync includes `NIFTY 50` and `NIFTY BANK` index references; these have
zero lot/tick sizes and are for dashboard inspection, not order placement.

## Kite market-data WebSocket

Paper and live runtimes wait for the initial Kite WebSocket handshake before
reporting the runtime as started. The client negotiates TLS 1.2 with Kite's
WebSocket endpoint. If startup reports a WebSocket error, check the structured
`KITE_ERROR` and `KITE_WS_DISCONNECTED` logs, confirm the daily Kite access
token is current, and verify outbound connectivity to `ws.kite.trade:443`.
This check does not place orders.

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
