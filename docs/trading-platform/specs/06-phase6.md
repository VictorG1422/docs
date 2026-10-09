# Phase 6 — Market Data & Technical Analysis Engine Foundation

> **Note on provenance:** the original chat attachment containing this
> phase's exact wording is not recoverable from session history (pasted
> attachment bodies are not persisted in the local debug transcript --
> only a placeholder reference is). This file is reconstructed from the
> phase's actual, verified implementation (see the "Phase 6" section of
> [README.md](../README.md) and `git log`) so the `queries/` record stays
> complete across phases. If you have the original prompt text, replace
> this file with it for an exact record.

Phase 1 through Phase 5 have been completed.

Phase 5 implemented underlying discovery, expiry discovery, strike
discovery, the option contract model, option-chain representation and
lookup, and optional Redis option-chain caching, all on top of the
Phase 4 broker/tick-pipeline foundation.

Now implement PHASE 6 ONLY.

The objective of Phase 6 is to build candle aggregation, a historical
market-data interface, and a technical-indicator engine on top of the
Phase 4 tick pipeline.

IMPORTANT:

Do NOT implement trading strategy.

Do NOT implement signal generation.

Do NOT decide which contract should be traded.

Do NOT implement order placement.

Do NOT implement risk management.

Do NOT implement live trading.

Do NOT implement Phase 7 or later phases.

This phase answers: "What is the current market state and what
technical information can we derive from it?"

## 1. Inspect Existing Project First

Before making any changes:

- Inspect the complete repository.
- Review Phase 1 through Phase 5 architecture.
- Review the existing tick processor, `MarketState`, `MarketStateStore`,
  and instrument/option-chain services.
- Review existing tests.
- Reuse the existing architecture. Do not duplicate configuration,
  logging, database, Redis, or instrument/option-chain discovery.
- Do not rewrite working code unnecessarily.

Before implementation, briefly explain:

1. How ticks will be aggregated into candles across multiple
   timeframes.
2. How the current (forming) candle differs from a completed candle.
3. How historical market data will be retrieved.
4. Which indicators will be implemented and how precision will be
   handled.
5. Which files will be created.
6. Which files will be modified.

Then implement Phase 6.

## 2. Tick Validation

Extend tick validation (on top of the existing Phase 4 checks) to
reject additional obviously malformed ticks (e.g. non-positive price,
negative volume/open-interest, or an internally inconsistent OHLC
snapshot) before they update state or reach any consumer.

## 3. Candle Aggregation

Aggregate validated ticks into OHLCV candles for one or more configured
timeframes.

Support querying the current (still-forming) candle separately from a
bounded history of completed candles per instrument+timeframe.

Define candle boundaries deterministically and document the choice.

Handle the fact that broker tick volume is typically a cumulative
session total, not a per-tick delta, without over-counting volume.

## 4. Historical Market Data

Provide a clean interface to retrieve historical OHLCV data for an
instrument/timeframe/date-range from the broker, isolated behind the
existing broker abstraction -- never called directly from outside it.

## 5. Technical Indicator Engine

Implement commonly used technical indicators as pure functions over
candle history (e.g. moving averages, momentum/oscillator indicators,
volatility indicators, volume-weighted price). Do not use a strategy
here to decide which indicators are "needed" -- implement general-
purpose indicator building blocks only.

Every indicator must:

- Return a clear "not enough data" result instead of raising or
  fabricating a value during warm-up.
- Use precise (non-lossy) arithmetic for anything that will be compared
  or persisted.

Do not duplicate an indicator calculation outside this engine.

## 6. Storage

Persist only completed candles (never the forming one) to PostgreSQL,
with protection against duplicate rows for the same
instrument+timeframe+timestamp.

Cache only the latest completed candle per instrument+timeframe in
Redis as a performance optimization -- PostgreSQL remains authoritative.

## 7. Market-Data Quality and Staleness

Reuse the existing Phase 4 staleness/health concepts -- do not invent a
second, separate staleness mechanism for candles.

## 8. Testing

Create deterministic unit tests for: tick validation changes, candle
aggregation (bucketing, volume handling, current vs. completed candle
handling, duplicate/out-of-order ticks), every indicator (valid data,
insufficient data, known calculations), historical data retrieval, and
the storage/cache layer.

Create integration tests covering the full pipeline against real local
PostgreSQL/Redis, gated behind an opt-in environment variable.

## 9. Definition of Done

Phase 6 is complete only when:

- Tick validation extended.
- Candle aggregation implemented for one or more configurable
  timeframes, with current vs. completed candle handling.
- Historical market-data interface implemented behind the broker
  abstraction.
- Technical indicator engine implemented with precise arithmetic and
  explicit insufficient-data handling.
- Completed candles persisted to PostgreSQL with duplicate protection.
- Redis caching implemented for the latest completed candle only.
- Market-data staleness handling reused, not duplicated.
- Tests implemented and passing.
- Phase 1 through Phase 5 tests still pass.
- README updated.
- No strategy logic, signal generation, or order placement implemented.

## Final Report

At the end of implementation, report: architecture explanation, files
created, files modified, test results, Phase 1-5 regression results,
architectural decisions, and known limitations/TODOs for Phase 7+.

IMPORTANT: STOP AFTER PHASE 6. Do not implement: trading strategy,
signal generation, option-chain selection for trading, order placement,
stop loss, targets, risk management, or live trading.
