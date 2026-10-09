# Phase 6 — Market Data & Technical Analysis Engine Foundation

Phase 6 builds candle aggregation, historical OHLCV retrieval, and
technical indicators on top of the Phase 4 tick pipeline -- it never
generates trading signals or makes strategy decisions; that remains
Phase 7+.

**Market-data flow:**

```
Kite WebSocket -> TickProcessor -> MarketState (Phase 4, unchanged)
                        │
                        ▼
              MarketDataService.on_tick()
                        │
                        ▼
     one CandleAggregator per configured interval (1/5/15 min by default)
                        │  on bucket rollover
                        ▼
        completed Candle -> PostgreSQL (CandleRepository)
                          -> Redis (CandleCacheStore, latest completed only)
```

- `trading_system/market_data/market_data_service.py::MarketDataService`
  is the single facade later phases read market data through. It wraps
  `MarketState` (Phase 4), one `CandleAggregator` per interval,
  `CandleRepository` (PostgreSQL), `CandleCacheStore` (Redis), and
  `HistoricalMarketDataProvider` -- without duplicating any of them.
  `on_tick()` is registered with `TickProcessor.add_listener` in
  `Application.__init__`, so candle aggregation runs off the exact same
  live tick stream as everything else (no second WebSocket subscription).

**Tick processing (validation tightened, not replaced):**

- `market_data/tick_processor.py::_is_valid_tick` now also rejects
  zero/negative `last_price` (previously only negative), negative
  `volume`/`oi`, and an impossible OHLC relationship (`high < open`,
  `low > close`, etc., via `_is_valid_ohlc`) whenever a tick carries a
  full OHLC snapshot. Malformed ticks are dropped before they ever reach
  `MarketState`, Redis, or a `CandleAggregator` -- never silently
  corrected.

**Candle aggregation:**

- `market_data/candle_aggregator.py::CandleAggregator` maintains one
  *forming* candle and a bounded list of *completed* candles per
  instrument, for a single fixed interval. `on_tick()` is the only entry
  point; a candle is marked complete only when a tick for the *next*
  bucket arrives -- there is no wall-clock timer, so the process needs
  no background thread, but the last candle of a session stays "forming"
  until the next session's first tick (session finalization is a later
  concern).
  - **Candle boundaries are epoch-aligned, not market-open-aligned**:
    each tick's UTC timestamp is floored to
    `epoch_seconds // (interval_minutes * 60)`. This is deliberately
    simple, uniform across every interval, and DST-proof; a later phase
    can layer market-open-aligned boundaries on top if a strategy
    specifically needs them.
  - **Volume-delta handling**: Kite's tick `volume` is the exchange's
    *cumulative* session volume, not a per-tick delta. `CandleAggregator`
    tracks a running per-instrument baseline and only accumulates the
    delta since the previous tick; a counter that goes backwards (e.g. a
    new session) is treated as a zero delta rather than corrupting the
    candle with a huge/negative number.
  - Late/out-of-order ticks for an already-rolled-over bucket are
    dropped (logged), never applied retroactively to a candle that may
    already be completed and handed to listeners.
  - `add_completion_listener()` registers a callback invoked on every
    candle completion; a listener exception is caught and logged, never
    allowed to break aggregation for other listeners or instruments.
- `trading_system/models/candle.py::Candle` is the aggregated OHLCV
  domain model (mutable, unlike other Pydantic models here, since the
  forming candle is updated in place): `open`/`high`/`low`/`close` are
  `Decimal` (never `float` -- exact precision preserved for anything
  persisted or aggregated), `volume`/`open_interest` are `int`,
  `timestamp` must be timezone-aware (validated) and `is_complete`
  distinguishes a forming candle from a completed one.

**Historical market data:**

- `broker/kite_client.py::KiteBroker.get_historical_data()` wraps
  `kiteconnect`'s `historical_data()` call, mapping `interval_minutes`
  (1/5/15/30/60) to Kite's interval strings and translating any
  `kiteconnect.exceptions.*` failure the same way every other broker
  call does. Deliberately **not** added to the `Broker` ABC -- doing so
  would force every existing `FakeBroker`/test double across the
  codebase to implement it, for a method only `HistoricalMarketDataProvider`
  currently calls.
- `market_data/historical_data.py::HistoricalMarketDataProvider` is a
  clean interface Phase 7+ can consume without touching the Kite SDK
  directly: `get_candles(instrument_token, start, end, interval_minutes)`
  normalizes broker rows into `Candle` objects and raises the typed
  `HistoricalDataError` on failure. Uses structural `Protocol` typing
  (not inheritance) for its broker dependency, so any object with a
  matching `get_historical_data` method works, including test doubles.
  Never caches -- historical ranges are typically large and requested
  infrequently, unlike the hot latest-tick/candle paths.

**Technical indicators (pure functions, no strategy logic):**

- `market_data/indicators.py` implements SMA, EMA, RSI (Wilder's
  smoothing), MACD (+ signal + histogram), ATR, VWAP, and Bollinger
  Bands, all hand-written (no third-party TA dependency) and operating
  on a `list[Candle]` (oldest-to-newest, typically
  `CandleAggregator.get_completed_candles()`). Every function returns
  `None` (never raises, never fabricates a value) when there isn't
  enough history -- insufficient data during warm-up is an expected
  condition, not an error. All arithmetic uses `Decimal` throughout,
  including `Decimal.sqrt()` for Bollinger Bands' standard deviation, so
  no candle value is ever coerced through binary `float`. These
  functions never decide entry/exit -- Phase 7 consumes their output to
  make that decision.

**PostgreSQL vs Redis responsibilities (unchanged split, extended):**

- **PostgreSQL** (`storage/db/models.py::Candle`, `storage/db/repositories.py::CandleRepository`)
  is the durable, authoritative store for *completed* candles only --
  never the forming one. `candles` has a unique constraint on
  `(instrument_token, interval_minutes, timestamp)`; `CandleRepository.save_if_new()`
  pre-checks for an existing row (on top of the DB constraint) so
  re-processing the same tick stream never raises a failed-insert error.
  `migrations/versions/0004_candles.py` creates the table.
- **Redis** (`storage/redis/candle_cache.py::CandleCacheStore`, key
  `RedisKeyBuilder.candle(instrument_token, interval_minutes)`) caches
  only the **latest completed** candle per instrument+interval (TTL 300s
  default) -- a cross-process performance read, never written per-tick.
  A cache miss or `RedisUnavailableError` returns `None`, which
  `MarketDataService` never confuses with "no candle exists yet".

**Data precision and timezone handling:**

- Every `Candle` field that is persisted, aggregated, or fed into an
  indicator is `Decimal` -- never `float` -- so no rounding error can
  accumulate across candles/indicators. (Legacy models `Tick`/`Instrument`
  are intentionally left as `float`/untouched -- Phase 6 does not rewrite
  working Phase 1-5 code.)
- `utils/time_utils.py::to_utc()` is the single place naive-vs-aware
  timestamp ambiguity is resolved: a naive `datetime` is assumed to
  already be IST (matching what Kite's WebSocket ticks and historical-data
  rows actually contain) and converted to a UTC-aware `datetime`; an
  already-aware `datetime` is simply converted to UTC. Every timestamp
  that crosses a candle/indicator/historical-data boundary goes through
  `to_utc()` first.

**Market-data staleness:** unchanged from Phase 4 --
`MarketDataService.get_data_status()` delegates to the existing
`MarketState.status()` (`FRESH`/`STALE`/`UNAVAILABLE`); Phase 6 does not
introduce a second staleness concept for candles, since a forming candle
is only ever as fresh as its instrument's latest tick.

**Testing:** `tests/unit/test_candle_aggregator.py` (bucketing
determinism, volume-delta/reset handling, late-tick rejection, candle
completion + listener isolation, multi-instrument independence),
`tests/unit/test_indicators.py` (one set of cases per indicator: valid
data, insufficient data, known deterministic calculations, output
structure), `tests/unit/test_historical_data.py` (row mapping, missing
optional fields, naive-timestamp handling, error translation),
`tests/unit/test_market_data_service.py` (tick fan-out, persistence,
caching, cache-outage fallback, historical-data delegation), plus
extensions to `tests/unit/test_repositories.py` (`CandleRepository`),
`tests/unit/test_redis_stores.py`/`test_redis_keys.py` (`CandleCacheStore`,
the `candle()` key), `tests/unit/test_time_utils.py` (`to_utc()`),
`tests/unit/test_tick_processor.py` (the tightened validation rules), and
`tests/unit/test_kite_client.py`/`test_application.py`
(`get_historical_data`, `MarketDataService` wiring). All use in-memory
SQLite/`fakeredis`/fake brokers -- never real Postgres/Redis/Kite.

See the original prompt: [specs/06-phase6.md](../specs/06-phase6.md).
