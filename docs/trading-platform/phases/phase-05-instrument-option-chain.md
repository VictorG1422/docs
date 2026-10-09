# Phase 5 — Instrument Discovery & Option Chain Foundation

Phase 5 builds a reusable instrument-discovery and option-chain layer on
top of the instrument data already synchronized into PostgreSQL by the
Phase 4 `InstrumentSyncService` -- it never downloads from Kite for a
lookup. It intentionally does **not** decide *which* contract should be
traded, implement any strategy logic, or generate signals; that remains
Phase 6+.

**Instrument discovery flow:**

```
Kite -> Instrument Sync -> PostgreSQL -> Instrument Repository -> Discovery Services
```

- `trading_system/market_data/underlying_discovery.py::UnderlyingDiscoveryService`
  resolves an underlying symbol (`NIFTY`, `BANKNIFTY`, ...) to an
  `UnderlyingInstrument` (exchange/segment/instrument_type/token metadata).
  Kite's instrument dump only has a distinct index-ticker row for the raw
  underlying, which Phase 4's sync intentionally filters out (only
  `EQ`/`FUT`/`CE`/`PE` contracts are kept); the most reliable single
  instrument left to represent an underlying is therefore its
  earliest-expiry futures contract when one has been synced, falling
  back to `instrument_token=None` if only options exist rather than
  guessing an arbitrary CE/PE row. `SUPPORTED_UNDERLYINGS` (currently
  `NIFTY`/`BANKNIFTY`) is a plain, extensible set -- adding another
  underlying never requires touching the discovery architecture.
- `trading_system/market_data/expiry_service.py::ExpiryService` reads
  actual expiry dates from synchronized instruments -- it never assumes
  a fixed weekly/monthly schedule. Provides `get_available_expiries`,
  `get_expiries_between`, `get_nearest_expiry`, and `get_next_expiry`
  (both expiry-selection methods use the application's IST clock,
  `utils.time_utils.now_ist()`, so an expired contract is never returned
  as "nearest").
- `trading_system/storage/db/repositories.py::InstrumentRepository`
  gains discovery queries (`get_by_exchange`, `list_by_underlying`,
  `get_any_future`, `list_expiries`, `list_strikes`,
  `list_options_for_expiry`, `list_options`) -- all `SELECT`s scoped to
  the injected `Session`, no new database layer.

**Option-chain representation and lookup:**

```
Underlying -> Expiry -> Strike -> CE / PE
```

- `trading_system/models/option_contract.py::OptionContract` -- an
  immutable snapshot of a single contract's metadata
  (`instrument_token`/`exchange_token`/`tradingsymbol`/`underlying`/
  `expiry`/`strike`/`option_type`/`exchange`/`segment`/`lot_size`/
  `tick_size`). `strike`/`tick_size` are `Decimal` (matching the ORM
  column type), never `float` -- exact precision is never lost.
- `trading_system/market_data/option_chain_service.py::OptionChainService`
  is the facade later phases use: `get_available_expiries`,
  `get_available_strikes`, `get_call_contract`/`get_put_contract`/
  `get_contract` (deterministic -- exactly one match or a domain error),
  and `get_option_chain` (a full `OptionChain` of `OptionChainStrike`
  entries, each holding the call and/or put contract at that strike).
  Never places an order or decides which contract to trade.
- Contract lookup is deterministic: 0 matches raises
  `ContractNotFoundError`; more than 1 match (a data-quality issue,
  never expected in practice) raises `DuplicateContractError` rather
  than silently returning an arbitrary contract. Malformed instrument
  records (missing/invalid strike, expiry, or option type) raise
  `InvalidInstrumentDataError` instead of being silently normalized.

**Caching (Redis is a performance cache only):**

- `trading_system/storage/redis/option_chain_cache.py::OptionChainCacheStore`
  caches a full `get_option_chain()` result, keyed by
  `RedisKeyBuilder.option_chain(underlying, expiry)` (namespaced,
  environment-scoped, same convention as every other Phase 3 key), TTL'd
  (60s default). **PostgreSQL remains authoritative** -- a cache miss
  (including a `RedisUnavailableError`) always falls through to a
  database query; a Redis outage is logged (`OPTION_CHAIN_CACHE_MISS`)
  and never mistaken for "contract not found." Single-contract lookups
  (`get_contract`/`get_call_contract`/`get_put_contract`) are not
  cached -- only the heavier full-chain query is.
- `config/settings.py`/`Application` are untouched here: caching reuses
  the existing `RedisClient`/`RedisStateManager`/`RedisKeyBuilder`
  (Phase 3) -- no second Redis client.

**Database changes:**

- `migrations/versions/0002_instrument_discovery_indexes.py` adds
  `ix_instruments_name` (single column) and
  `ix_instruments_name_type_expiry` (composite on
  `name, instrument_type, expiry`) -- the discovery hot path always
  filters by underlying + instrument type + expiry together. No index
  was added on `exchange`/`segment`: cardinality is a handful of
  distinct values table-wide, and every query already filters by `name`
  first, so those columns wouldn't meaningfully narrow the row set.
- `migrations/versions/0003_instrument_exchange_token.py` adds a
  nullable `exchange_token` column to `instruments` -- Phase 4's sync
  read `exchange_token` from Kite's dump but never persisted it;
  `OptionContract` needs the real value, not a stand-in. Nullable so
  existing rows don't need a backfill; re-running `sync_instruments()`
  populates it going forward.

**Health check:** `Application` registers an `"instrument_data"` health
check (`instrument_data: true` once at least one instrument row exists)
alongside `postgresql`/`redis`/`kite` -- a missing/not-yet-run sync is
reported as a meaningful `false`, not a crash.

**Design decisions / Phase 6+ TODOs:**

- **Nothing decides which contract to trade.** `OptionChainService`
  only answers "what's available" -- ATM/ITM/OTM selection, expiry-cycle
  policy (weekly vs. monthly), and entry/exit logic are explicit
  Phase 6+ strategy decisions.
- **Whole-chain duplicate strikes are logged and deduped (first-seen
  wins), not raised** -- unlike `get_contract`'s deterministic lookup,
  failing an entire chain over one bad row would be disproportionate;
  see `OptionChainService._dedup_by_strike`.
- **Testing**: `test_underlying_discovery.py`, `test_expiry_service.py`,
  and `test_option_chain_service.py` use an in-memory SQLite-backed
  `PostgresClient` (never real Postgres). `test_option_chain_service.py`
  also uses a fake cache double to test the cache-hit/cache-unavailable
  paths without `fakeredis`. `tests/integration/test_kite_market_data_integration.py`
  gains `Kite/mock broker -> Instrument Sync -> PostgreSQL -> Instrument
  Repository -> Option Chain Service` (real Postgres/Redis, gated behind
  `RUN_INTEGRATION_TESTS=1`); it now also relies on Alembic-applied
  schema and cleans up only the specific rows it creates instead of
  calling `Base.metadata.create_all()`/`drop_all()` against the
  configured database.

See the original prompt: [specs/05-phase5.md](../specs/05-phase5.md).
