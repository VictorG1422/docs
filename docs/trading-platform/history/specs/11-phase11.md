# Phase 11 — Backtesting & Paper Trading Engine

## Project Roadmap

**Previously Completed Phases:**
- Phase 1 — Core Application Foundation
- Phase 2 — PostgreSQL Database
- Phase 3 — Redis State & Cache Layer
- Phase 4 — Kite Connect & Market Data Foundation
- Phase 5 — Instrument & Option-Chain Foundation
- Phase 6 — Market Data & Technical Analysis
- Phase 7 — Trading Strategy & Signal Engine
- Phase 7A — Actual Strategy Specification
- Phase 8 — Risk & Position Management
- Phase 9 — Order Execution Engine
- Phase 10 — Broker Reconciliation & State Consistency

**Current Phase:** Phase 11 — Backtesting & Paper Trading Engine

**Upcoming Phases:**
- Phase 12 — Production & Live-Trading Readiness
- Phase 13 — Performance Optimization & Monitoring

**IMPORTANT:** Implement Phase 11 ONLY. Inspect the existing project and reuse its architecture. Do not assume earlier phases are correct merely because they were marked complete; verify their implementation and tests.

Do not implement Phase 12 or Phase 13.

## 1. Phase 11 Objective

Build a reliable framework to evaluate the existing trading strategy without placing real broker orders.

Phase 11 must support two distinct modes:

**A. Historical Backtesting** — Run the existing strategy against historical market data and simulate order execution and portfolio changes.

**B. Paper Trading** — Process live or recorded market data through the existing strategy, risk engine, and paper execution adapter without submitting real orders to Kite.

Expected architecture:

```
Historical/Recorded/Live Market Data
  ↓
Existing Market Data & Indicator Layer
  ↓
Existing Strategy Engine
  ↓
Existing Risk & Position Management
  ↓
Backtest or Paper Execution Adapter
  ↓
Simulated Order Lifecycle
  ↓
Simulated Positions and P&L
  ↓
PostgreSQL Results and Reports
```

Do not duplicate strategy rules, indicators, risk calculations, or signal generation.

## 2. Inspect the Complete Project First

Before modifying code, inspect the repository and review:
- Phase 1 configuration, logging, and health checks.
- Phase 2 PostgreSQL models and repositories.
- Phase 3 Redis state and locking.
- Phase 4 Kite integration and market-data interfaces.
- Phase 5 instrument and options metadata.
- Phase 6 indicators and market-data processing.
- Phase 7 strategy and signal generation.
- Phase 7A actual strategy conditions and parameters.
- Phase 8 risk checks, approved TradeIntent, and position sizing.
- Phase 9 order interfaces, execution records, and lifecycle.
- Phase 10 reconciliation and state consistency.

Identify the actual strategy entry and exit conditions, configured instruments, candle intervals, signal format, order types, fees configuration, and risk parameters.

Do not invent missing strategy conditions.

If essential strategy rules or data requirements are missing, document them and ask for clarification instead of inventing them.

Before implementation, briefly explain:
1. Existing architecture.
2. Strategy and signal flow.
3. Historical data availability.
4. How backtesting will reuse the existing strategy.
5. How paper execution will remain isolated from live execution.
6. Files to create and modify.
7. Testing approach.

Then implement Phase 11.

## 3. Strict Safety Boundary

Phase 11 is simulation only. It must never:
- Place real orders through Kite.
- Modify or cancel real broker orders.
- Change strategy rules without explicit authorization.
- Automatically enable live execution.
- Use a production broker account for tests.
- Change production positions.
- Treat simulated fills as real broker fills.
- Write paper positions into production trading state.
- Automatically promote a strategy to live trading.

The existing Phase 9 execution engine must remain protected by its explicit live-execution safety gate.

Paper trading must use a separate execution adapter and isolated state.

## 4. Backtest Engine

Create a reusable BacktestEngine. It should accept:
- Strategy configuration and version.
- Historical market data.
- Instrument metadata.
- Candle interval.
- Start and end timestamps.
- Initial simulated capital.
- Risk configuration.
- Execution simulation settings.
- Brokerage and transaction-cost settings.

Conceptual flow:

```
Backtest Configuration
  ↓
Historical Data Loader
  ↓
Chronological Data Feed
  ↓
Existing Indicator and Strategy Engine
  ↓
Existing Risk Engine
  ↓
Simulated Execution
  ↓
Portfolio Accounting
  ↓
Performance Report
```

Use the existing architecture and avoid introducing unnecessary frameworks.

## 5. Historical Data

Inspect whether historical data is already available through the existing Kite integration or local data storage. Reuse existing interfaces wherever possible.

If historical data must be retrieved through Kite:
- Use the existing Kite client.
- Respect API limits.
- Use only supported historical-data endpoints.
- Handle missing data and API failures.
- Avoid unnecessary downloads.
- Cache reusable historical data where appropriate.
- Record the source and retrieval time.

Do not assume historical data contains every required option contract, tick, candle, or market event. Do not fabricate missing prices or candles.

If the required dataset is unavailable, report the limitation and provide a clean data-provider interface for later use.

Use an explicit data schema that identifies:
- Instrument token.
- Exchange and trading symbol.
- Timestamp and timezone.
- Open.
- High.
- Low.
- Close.
- Volume.
- Open interest, when available.

Validate timestamps, ordering, duplicates, missing intervals, and invalid OHLC values.

Prevent look-ahead bias: the strategy must never access future data while making a decision at the current simulated timestamp.

## 6. Chronological Simulation

Process data in chronological order. At every simulation step:
1. Advance the simulated market clock.
2. Make only data available at that time accessible to the strategy.
3. Update indicators using the existing indicator implementation.
4. Evaluate the existing strategy.
5. Pass generated signals through the existing risk engine.
6. Simulate execution only for eligible, approved intents.
7. Update simulated positions, cash, fees, and P&L.
8. Record relevant events and state changes.

Do not execute trades retrospectively at prices that were unknown when the signal occurred.

Clearly define when a signal becomes available and the earliest permissible execution price.

## 7. Look-Ahead and Data-Leakage Protection

Implement safeguards against:
- Using future candles.
- Using future indicator values.
- Executing at a price observed before the signal existed.
- Using future option-chain information.
- Using revised data unavailable at the time.
- Calculating indicators from incomplete candles when the strategy requires completed candles.

Add tests proving that changing future market data does not change decisions made at earlier timestamps.

Document assumptions about candle-close signals and next-candle execution.

## 8. Simulated Order Execution

Create a dedicated PaperExecutionAdapter or equivalent simulation component. It must not call the real Kite order-placement API.

Support only order types required by the existing strategy.

Simulate appropriate order lifecycle states, such as: CREATED, OPEN, PARTIALLY_FILLED (where applicable), FILLED, CANCELLED, REJECTED.

Use explicit execution assumptions.

For market orders, define which available price is used and how slippage is applied.

For limit orders, do not assume a fill merely because a candle's high or low touched the limit price. Account for the limits of OHLC data and document the chosen conservative fill model.

For stop-loss or trigger orders, simulate trigger and execution behavior without assuming guaranteed fills at the trigger price.

If intrabar price ordering cannot be determined from available candle data, apply a documented conservative rule or use higher-resolution data when available.

Do not implement a misleading execution model that produces unrealistically favorable results.

## 9. Partial Fills and Liquidity

Where the data and simulation model support it, allow: partial fills, remaining quantity, average simulated fill price, slippage, spread assumptions, insufficient simulated liquidity.

Do not claim realistic market-impact modelling when the available data cannot support it.

Document the difference between simulated fills and actual broker fills.

## 10. Brokerage and Transaction Costs

Implement configurable transaction-cost calculations using the existing project configuration where possible.

Consider applicable costs, including: brokerage, exchange transaction charges, securities transaction tax where applicable, GST where applicable, SEBI charges, stamp duty where applicable, slippage, bid-ask spread assumptions.

Do not hardcode a supposedly universal fee schedule. Rates may depend on exchange, instrument, product, order type, and current rules.

Make rates configurable and versioned. Do not invent current fee values. If current rates have not been verified, require explicit configuration and clearly mark defaults as unverified.

Report gross P&L separately from net P&L.

## 11. Paper Portfolio and Position Accounting

Create an isolated simulated portfolio. Track:
- Initial capital.
- Available simulated cash.
- Used margin, where supported.
- Open positions.
- Position quantity.
- Average simulated entry price.
- Realized P&L.
- Unrealized P&L.
- Fees and charges.
- Net P&L.
- Equity.
- Peak equity.
- Drawdown.

Use Decimal or an appropriate exact numeric representation for financial values.

Use explicit rules for averaging into positions, reducing positions, closing positions, and handling reversals.

Reuse the existing risk and position models where appropriate, but do not mutate production position state.

If the existing accounting rules are incomplete, document the gap instead of silently inventing behavior.

## 12. Strategy and Risk Reuse

Use the existing Phase 7/7A strategy and Phase 8 risk engine. Do not implement a second version of the strategy.

Do not alter: entry conditions, exit conditions, indicator definitions, position-sizing logic, risk limits, stop-loss logic, target logic.

Allow explicitly configured experimental parameters only if the existing architecture supports them. Record the complete configuration and strategy version for every run.

A backtest must not bypass risk checks simply to generate more trades.

## 13. Backtest Configuration

Create a validated BacktestConfig containing the required settings. Include, as appropriate:
- Unique run ID.
- Environment.
- Strategy name and version.
- Strategy parameters.
- Instrument identifiers.
- Candle interval.
- Historical date range.
- Initial capital.
- Risk configuration.
- Cost model.
- Execution model.
- Data source.
- Random seed if any stochastic simulation is used.

Reject invalid date ranges, missing parameters, unsupported instruments, invalid capital, and incompatible data intervals.

Make backtests reproducible from their saved configuration and input-data version.

## 14. Backtest Results

Create structured results containing, where meaningful:
- Run ID.
- Strategy name/version.
- Start and end timestamps.
- Initial and final equity.
- Gross profit and loss.
- Net profit and loss.
- Total return.
- Number of trades.
- Winning trades.
- Losing trades.
- Win rate.
- Average win.
- Average loss.
- Profit factor.
- Maximum drawdown.
- Average holding time.
- Total transaction costs.
- Exposure.
- Rejected trade intents.
- Data-quality warnings.

Define every metric clearly.

Handle zero trades, zero losses, and insufficient observations without division-by-zero errors or misleading infinite values.

Do not label a strategy profitable solely based on win rate.

## 15. Risk and Performance Metrics

Where the available data supports them, calculate: maximum drawdown, return volatility, Sharpe ratio, Sortino ratio, risk-to-reward statistics, consecutive wins and losses, exposure-adjusted returns.

Document the observation frequency, risk-free-rate assumption, annualization method, and minimum sample requirements.

Return unavailable metrics as unavailable rather than manufacturing values from insufficient data.

Do not treat historical performance as a guarantee of future results.

## 16. Trade-by-Trade Audit

Persist enough information to reproduce and investigate every simulated trade. Include:
- Run ID.
- Signal ID.
- TradeIntent ID, where applicable.
- Instrument.
- Strategy version.
- Signal timestamp.
- Simulated submission timestamp.
- Fill timestamps.
- Entry and exit prices.
- Quantity.
- Gross P&L.
- Transaction costs.
- Net P&L.
- Exit reason, if provided by the existing strategy.
- Simulated order lifecycle.
- Relevant risk decision.
- Execution-model assumptions.

Do not invent entry or exit reasons.

Preserve rejected intents and execution failures for analysis.

## 17. PostgreSQL Persistence

Use the existing Phase 2 database architecture. Persist: backtest runs, configuration snapshots, data-source and dataset metadata, simulated orders, simulated fills, simulated trades, performance summaries, run errors and warnings.

Use appropriate transactions and indexes. Ensure each run and its records are identifiable and isolated.

Do not store simulated orders in the production broker-order table in a way that could be mistaken for real orders.

If existing tables are shared, explicitly distinguish execution environment and order origin.

Do not delete existing production records.

## 18. Redis Isolation

Reuse Phase 3 Redis infrastructure. Use separate namespaces or the project's existing environment isolation to distinguish: development, paper trading, production.

Paper state must never overwrite production runtime state.

Do not store the only copy of durable backtest results in Redis.

Avoid scanning the entire Redis keyspace.

## 19. Paper Trading Mode

Implement a paper-trading service that can consume either: recorded market data, live market data from the existing Phase 4 market-data interface.

The service must use the same strategy and risk pipeline as intended for live trading, while routing all simulated orders to PaperExecutionAdapter.

Architecture:

```
Market Data
  ↓
Existing Indicators
  ↓
Existing Strategy
  ↓
Existing Risk Engine
  ↓
Paper Execution Adapter
  ↓
Paper Portfolio
  ↓
PostgreSQL Paper Records
```

Live market data does not mean live execution. No Kite order-placement calls are permitted in paper mode.

If market data becomes stale or unavailable, apply an explicit safe policy and prevent simulated fills from using stale prices without a warning.

## 20. Paper-Trading Session Management

Support controlled paper sessions with: session ID, start time, stop time, strategy version, configuration snapshot, selected instruments, session status, last processed market timestamp, error status, summary metrics.

Support safe start, stop, and resume behavior where the existing architecture permits it.

Prevent accidental duplicate paper sessions for the same configured session identity.

Do not create an uncontrolled infinite loop or unmanaged background worker.

## 21. Restart and Recovery

Ensure backtests and paper sessions handle interruptions safely.

For backtests, store enough metadata to identify incomplete runs and determine whether they can be restarted.

For paper sessions, restore only paper-specific state from durable records and available market data.

Do not automatically resume live execution. Do not assume Redis state survived a restart.

Prevent duplicate simulated fills when replaying already processed data.

Use deterministic event identifiers or equivalent idempotency controls where appropriate.

## 22. Paper Mode Safety Gate

Create an explicit execution-mode distinction: BACKTEST, PAPER, LIVE.

BACKTEST and PAPER must use simulated execution. LIVE must remain governed by the existing Phase 9 safety gate.

Tests must prove that BACKTEST and PAPER cannot reach the real Kite order-placement API.

Do not change the live-execution enable flag or enable production orders.

## 23. API or CLI Integration

Follow the project's existing application interface. Provide a minimal supported way to:
- Start a backtest.
- Inspect backtest status.
- Retrieve a completed result.
- List recent runs.
- Start a paper session.
- Stop a paper session.
- Inspect paper-session status.
- Retrieve paper-session performance.

Use the existing API or CLI architecture. Do not add a web framework solely for this phase.

Validate user inputs and handle invalid requests clearly.

## 24. Logging and Error Handling

Reuse the Phase 1 logger. Add structured events such as: BACKTEST_STARTED, BACKTEST_COMPLETED, BACKTEST_FAILED, BACKTEST_DATA_WARNING, PAPER_SESSION_STARTED, PAPER_SESSION_STOPPED, PAPER_ORDER_CREATED, PAPER_ORDER_FILLED, PAPER_ORDER_REJECTED, PAPER_SESSION_ERROR.

Do not log every market-data tick at INFO level.

Handle: missing historical data, invalid candles, duplicate timestamps, indicator initialization, strategy exceptions, risk rejection, simulation failures, database errors, Redis errors, market-data disconnection, invalid configuration, interrupted runs.

Do not report a run as successfully completed if required processing failed.

Preserve actionable error details without logging credentials or secrets.

## 25. Testing — Backtest Engine

Create tests for: valid backtest configuration, invalid configuration, chronological processing, start and end date boundaries, empty dataset, missing candles, duplicate candles, invalid OHLC values, indicator warm-up, strategy signal handling, risk rejection, no-trade runs, reproducibility, backtest persistence, interrupted-run handling.

Use deterministic fixtures and synthetic datasets where appropriate.

## 26. Testing — Look-Ahead Bias

Create tests proving: future candles are unavailable to earlier decisions, indicators use only permitted data, a signal cannot fill at a price that was known only before the signal was generated, candle-close strategies execute according to the configured execution timing, missing or incomplete data is handled according to policy.

## 27. Testing — Simulated Execution

Test: market-order simulation, limit-order simulation, trigger-order simulation where required, invalid order request, partial fill, full fill, rejection, cancellation, slippage, fees, insufficient simulated liquidity, price precision, order lifecycle transitions.

Never use a real broker for these tests.

## 28. Testing — Accounting

Test: opening a simulated position, increasing a position, reducing a position, closing a position, realized P&L, unrealized P&L, average entry price, transaction costs, net P&L, equity updates, maximum drawdown, multiple instruments, multiple trades, rejected intents, zero-trade reports.

Verify calculations with hand-computed examples.

## 29. Testing — Paper Mode Safety

Use a fake broker/API boundary and instrumented adapters. Prove that: backtest orders use simulated execution, paper orders use simulated execution, real Kite order-placement methods are never called, paper state does not overwrite production state, duplicate events do not create duplicate simulated fills, a stopped session does not continue processing new events, resume behavior does not duplicate previously processed events.

## 30. Integration and Regression Tests

Create isolated integration tests using test PostgreSQL, test Redis, synthetic or fixture market data, and fake execution adapters.

Do not use production credentials or production broker orders.

Run existing tests for Phases 1–10. Do not weaken existing tests merely to make Phase 11 pass.

Report unavailable dependencies or tests that could not run honestly.

## 31. README Update

Update README with: Phase 11 architecture, backtesting setup, required historical data, dataset format, backtest configuration, strategy and risk reuse, simulated execution assumptions, fees and slippage configuration, performance metric definitions, reproducibility, paper-trading setup, paper-session management, PostgreSQL result storage, Redis isolation, restart and recovery, safety boundaries, known limitations, Phase 12 roadmap.

Document how to run a sample backtest using fixture data without requiring real orders or production credentials.

## 32. Security and Environment Isolation

Before completing Phase 11, verify: no hardcoded credentials, no secrets in logs, no production credentials required for tests, no real broker orders in backtests or paper sessions, production state is isolated from simulated state, paper sessions cannot accidentally enable live execution, historical datasets and configurations are identifiable, financial values use appropriate precision, no unsafe deserialization, no destructive database changes.

## 33. Definition of Done

Phase 11 is complete only when:
- Backtest engine implemented.
- Historical data interface implemented.
- Chronological simulation implemented.
- Look-ahead protections tested.
- Existing strategy reused.
- Existing risk engine reused.
- Paper execution adapter implemented.
- Simulated order lifecycle implemented.
- Portfolio accounting implemented.
- Transaction costs configurable.
- Performance metrics implemented.
- Trade-by-trade audit implemented.
- PostgreSQL persistence implemented.
- Redis isolation implemented.
- Paper session management implemented.
- Restart and duplicate-event handling implemented.
- Backtest and paper modes cannot place real orders.
- Unit tests implemented.
- Integration tests implemented where practical.
- Regression tests for Phases 1–10 pass.
- README updated.
- Known limitations documented.
- No Phase 12 or Phase 13 features implemented.

## 34. Final Report

At completion, report:
1. Existing architecture reviewed.
2. Files created.
3. Files modified.
4. Backtest architecture.
5. Historical data source and limitations.
6. Strategy and risk engine reuse.
7. Look-ahead bias protections.
8. Simulated execution assumptions.
9. Fee and slippage model.
10. Portfolio accounting.
11. Performance metrics.
12. PostgreSQL changes.
13. Redis changes.
14. Paper session management.
15. Restart and recovery behavior.
16. Confirmation that simulated orders cannot reach live Kite order-placement APIs.
17. Tests created and actual results.
18. Phase 1–10 regression results.
19. Security verification.
20. Known limitations and unresolved decisions.
21. Exact commands for running a sample backtest.
22. Exact commands for starting and inspecting paper trading.
23. TODOs for Phase 12.

**IMPORTANT:** STOP AFTER PHASE 11.

Do not implement Phase 12 production readiness. Do not enable live trading. Do not place real orders. Do not modify the strategy without authorization. Do not invent historical data or claim simulated results are real trading results.

**Final Flow:**

```
HISTORICAL DATA
  ↓
EXISTING STRATEGY
  ↓
EXISTING RISK ENGINE
  ↓
SIMULATED EXECUTION
  ↓
BACKTEST RESULTS
```

OR

```
LIVE/RECORDED MARKET DATA
  ↓
EXISTING STRATEGY
  ↓
EXISTING RISK ENGINE
  ↓
PAPER EXECUTION ONLY
  ↓
SIMULATED PORTFOLIO
```

STOP AFTER PHASE 11.

---

## Addendum — read-only dashboard (external idea, not part of the phase roadmap)

This was raised separately, after Phase 11 was already complete, as the
project owner's own idea for viewing the system's data -- explicitly not
"Phase 12" (that name is already reserved above for a later, unrelated
Production & Live-Trading Readiness phase). Folded into Phase 11 here
rather than given its own phase number.

> Inside trading platform repo, create frontend application to see required
> and production grade visible data from database. It just use jQuery and
> HTML/CSS, no more multiple languages, just simple frontend application.
> Must be connected with backend Python application that should deliver
> output of all required API calls from frontend. Create frontend and
> backend application and organize all folders in production standard.
> Then update all the documents.
>
> Follow-up: also render the frontend itself in Python (not only serve it
> as a static file) if possible.
