# Phase 5 — Instrument Discovery & Option Chain Foundation

> **Note on provenance:** the original chat attachment containing this
> phase's exact wording is not recoverable from session history (pasted
> attachment bodies are not persisted in the local debug transcript --
> only a placeholder reference is). This file is reconstructed from the
> phase's actual, verified implementation (see the "Phase 5" section of
> [README.md](../README.md) and `git log`) so the `queries/` record stays
> complete across phases. If you have the original prompt text, replace
> this file with it for an exact record.

Phase 1, Phase 2, Phase 3, and Phase 4 have been completed.

Phase 4 implemented the Kite Connect broker integration, instrument
synchronization into PostgreSQL, the WebSocket tick pipeline, and Redis
market-state integration.

Now implement PHASE 5 ONLY.

The objective of Phase 5 is to build a reusable instrument-discovery and
option-chain layer on top of the instrument data already synchronized
into PostgreSQL by Phase 4 -- it must never download from Kite for a
lookup.

IMPORTANT:

Do NOT decide which contract should be traded.

Do NOT implement any strategy logic.

Do NOT generate signals.

Do NOT implement order placement.

Do NOT implement risk management.

Do NOT implement live trading.

Do NOT implement Phase 6 or later phases.

This phase answers: "Which valid option contracts exist?" -- nothing
about which one should be traded.

## 1. Inspect Existing Project First

Before making any changes:

- Inspect the complete repository.
- Review Phase 1 through Phase 4 architecture.
- Review the existing `Instrument` model, `InstrumentRepository`, and
  the Phase 4 instrument-sync service.
- Review existing tests.
- Reuse the existing architecture.
- Do not create duplicate configuration, logging, database, or Redis
  systems.
- Do not rewrite working code unnecessarily.

Before implementation, briefly explain:

1. How underlying discovery will resolve a symbol (e.g. NIFTY) to
   synchronized instrument data.
2. How expiry discovery will work without assuming a fixed weekly/
   monthly schedule.
3. How option-contract lookup and the full option chain will be
   represented.
4. Which files will be created.
5. Which files will be modified.

Then implement Phase 5.

## 2. Underlying Discovery

Support at least NIFTY and BANK NIFTY.

Do not assume additional underlyings without extending an explicit,
documented set.

Resolve an underlying symbol to its representative synchronized
instrument using only PostgreSQL data already synced by Phase 4 -- never
re-download from Kite for this.

## 3. Expiry Discovery

Read actual expiry dates from synchronized instruments.

Do not assume a fixed weekly/monthly expiry schedule.

Support finding available expiries, expiries within a date range, and
the nearest/next expiry relative to the current IST trading clock so an
expired contract is never returned as "nearest".

## 4. Strike Discovery and CE/PE Filtering

Support listing available strikes for an underlying+expiry, and looking
up CE/PE contracts by strike.

Do not hardcode a strike-selection policy -- that is a later strategy
decision.

## 5. Option Contract Model

Create a clean, immutable representation of a single option contract's
metadata (instrument token, trading symbol, underlying, expiry, strike,
option type, exchange, segment, lot size, tick size).

Use exact-precision types (not floating point) for strike/tick size.

## 6. Option Chain Representation and Lookup

Create an option-chain abstraction:

```
Underlying -> Expiry -> Strike -> CE / PE
```

Provide deterministic contract lookup: zero matches is a "not found"
error, more than one match (a data-quality issue) is a distinct error --
never silently return an arbitrary contract.

Handle malformed instrument records (missing/invalid strike, expiry, or
option type) as invalid data, not silently normalized.

## 7. Caching

Redis is a performance cache only. PostgreSQL remains authoritative.

Cache a full option-chain lookup, keyed by underlying+expiry, with a
short TTL. A cache miss (including a Redis outage) must always fall
through to a database query.

Do not cache single-contract lookups if the full-chain cache already
covers the common case.

Reuse the existing Redis client/key-builder infrastructure -- do not
create a second Redis client.

## 8. Database Changes

Add any indexes required to make the discovery hot path
(underlying + instrument type + expiry) efficient.

If any field needed by the option-chain layer was read from Kite in
Phase 4 but never persisted, add the missing column via a migration.

Use Alembic migrations -- never manually created tables.

## 9. Health Check

Extend the existing health-check framework with an instrument-data
check: unhealthy (not crashed) if no instrument has been synchronized
yet.

## 10. Testing

Create unit tests for underlying discovery, expiry discovery, and
option-chain/contract lookup using an in-memory database -- never a real
Kite/Postgres/Redis connection.

Create at least one integration test covering
Kite/mock broker -> Instrument Sync -> PostgreSQL -> Instrument
Repository -> Option Chain Service, gated behind an opt-in environment
variable, using real local PostgreSQL/Redis.

## 11. Definition of Done

Phase 5 is complete only when:

- Underlying discovery implemented (NIFTY, BANK NIFTY).
- Expiry discovery implemented from real synchronized data.
- Strike discovery and CE/PE filtering implemented.
- Option contract model implemented with exact-precision fields.
- Option-chain representation and deterministic contract lookup
  implemented.
- Instrument validation and active/expired contract handling
  implemented.
- PostgreSQL indexes added for the discovery hot path.
- Optional Redis option-chain caching implemented, PostgreSQL remains
  authoritative.
- Tests implemented and passing.
- Phase 1 through Phase 4 tests still pass.
- README updated.
- No strategy logic, signal generation, or order placement implemented.

## Final Report

At the end of implementation, report: architecture explanation, files
created, files modified, test results, Phase 1-4 regression results,
architectural decisions, and known limitations/TODOs for Phase 6+.

IMPORTANT: STOP AFTER PHASE 5. Do not implement Phase 6 or later
functionality. Do not implement strategy logic, signal generation, order
placement, risk management, or live trading.
