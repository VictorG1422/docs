# Phase 7 — Trading Strategy & Signal Engine

## Project roadmap context

This trading system is being built in controlled phases.

### Completed phases

- **Phase 1 — Core Application Foundation.** Implemented the core application architecture, configuration, logging, health checks, project structure, and foundational services.
- **Phase 2 — PostgreSQL Database.** Implemented PostgreSQL configuration, database connectivity, durable persistence, models/repositories/migrations as required. PostgreSQL is the durable source of truth.
- **Phase 3 — Redis State & Cache Layer.** Implemented Redis infrastructure including: Redis configuration, connection pooling, health checks, a centralized Redis key builder, state management, JSON serialization, TTL support, market-state abstraction, strategy-state abstraction, position-state abstraction, distributed locks, duplicate-event protection, cache abstraction, Redis error handling, and tests. Redis is used for fast runtime state, caching, temporary state, locks, deduplication, and latest market state. Redis is NOT the durable source of truth.
- **Phase 4 — Kite Connect & Market Data Foundation.** Implemented: Kite configuration, authentication infrastructure, Kite client abstraction, Kite health check, instrument synchronization, instrument repository, Kite WebSocket, subscription manager, tick processor, Redis market-state integration, WebSocket reconnection, market-data staleness detection, broker error handling, and tests. Kite provides broker and market-data integration. No trading orders should be implemented in Phase 4.
- **Phase 5 — Instrument Discovery & Option Chain Foundation.** Implemented: underlying discovery, NIFTY support, BANK NIFTY support, expiry discovery, expiry filtering, strike discovery, CE/PE filtering, option contract model, option-chain representation, contract lookup, instrument validation, active/expired contract handling, PostgreSQL optimization/indexes, optional Redis option-chain caching, and tests. Phase 5 answers: "Which valid option contracts exist?"
- **Phase 6 — Market Data & Technical Analysis Engine.** Implemented: market-data abstraction, tick validation, candle aggregation, multiple timeframes, current/completed candle handling, historical market-data interface, market-data storage where required, market-data quality validation, technical indicator engine, indicator calculations, indicator precision handling, market-data staleness detection, Redis caching, and tests. Phase 6 answers: "What is the current market state and what technical information can we derive from it?"

Treat the existing Phase 1–6 implementation as the current baseline.

Do not rewrite working infrastructure unnecessarily.

Do not create duplicate systems.

### Remaining roadmap

- **Phase 7 — Trading Strategy & Signal Engine.** THIS PHASE. Build the actual strategy framework and generate structured trade signals.
- **Phase 8 — Risk & Position Management.** Will later implement: position sizing, risk limits, exposure limits, stop-loss, target, trade limits, risk validation, safety controls. Do NOT implement Phase 8 now.
- **Phase 9 — Order Execution Engine.** Will later implement: Kite order placement, order lifecycle, execution tracking, order retries, execution locks, broker execution handling. Do NOT implement Phase 9 now.
- **Phase 10 — Broker Reconciliation.** Will later reconcile: broker orders, broker positions, PostgreSQL, Redis, application state. Do NOT implement now.
- **Phase 11 — Backtesting & Paper Trading.** Will later validate the strategy without risking real capital. Do NOT implement now unless a tiny interface is required for Phase 7 testability.
- **Phase 12 — Production & Live-Trading Readiness.** Will later implement: production deployment, monitoring, alerts, recovery, audit, operational safeguards, controlled live rollout. Do NOT implement now.
- **Phase 13 — Optimization & Strategy Improvement.** Will later implement controlled strategy evaluation and optimization. Do NOT implement now.

## Phase 7 objective

Implement a clean, deterministic, testable trading strategy and signal-generation engine.

The strategy engine must consume:

- market data
- candles
- technical indicators
- instrument information
- option-chain information
- runtime strategy state

and produce:

- structured trading signals.

The strategy engine must NOT:

- place orders
- manage broker orders
- manage positions
- calculate final position size
- execute trades
- perform broker reconciliation
- enable live trading

Expected architecture:

```
Market Data -> Candle Data -> Technical Indicators -> Strategy Engine -> Signal Validation -> Signal -> Future Risk Engine
```

## Inspect existing project first

Before making changes:

- Inspect the complete repository.
- Review Phase 1.
- Review Phase 2.
- Review Phase 3.
- Review Phase 4.
- Review Phase 5.
- Review Phase 6.
- Review: configuration, logging, health checks, PostgreSQL, Redis, Kite client, WebSocket, tick processor, MarketStateStore, InstrumentRepository, OptionChainService, MarketDataService, CandleAggregator, IndicatorEngine, StrategyStateStore, tests, README.
- Reuse existing architecture.
- Do not duplicate: configuration, logging, database, Redis, market data, indicator calculations, instrument discovery, option-chain discovery.

Before implementation, briefly explain:

- Strategy architecture.
- Data flow into the strategy.
- Strategy state management.
- Signal structure.
- Signal validation.
- Duplicate signal handling.
- How Phase 8 will consume signals.
- Which files will be created.
- Which files will be modified.

Then implement Phase 7.

## Strategy design principle

The strategy must be modular.

Do NOT put the entire strategy into one giant function.

Create a clean abstraction such as: `Strategy` or `BaseStrategy`.

A strategy should conceptually receive market context and produce a signal or no signal. For example:

```
MarketContext -> Strategy -> StrategyDecision -> Signal
```

The architecture must allow another strategy to be added later without rewriting the entire application.

## Market context

Create a structured market context. It may contain:

- underlying
- instrument_token
- timestamp
- current_price
- candles
- timeframe
- technical indicators
- option-chain metadata
- strategy state

Only include information that is actually required. Do not create unnecessarily huge objects.

The strategy must not directly access:

- raw Redis commands
- raw PostgreSQL connections
- Kite SDK objects

Use existing application abstractions.

## Strategy configuration

Do not hardcode strategy parameters throughout the code.

Create configuration for strategy parameters where appropriate.

Potential parameters may include:

- timeframe
- indicator periods
- thresholds
- cooldown
- confirmation requirements

The exact parameters must be derived from the project's intended strategy requirements.

If the existing project specification already defines strategy parameters, use those.

Do not invent arbitrary profitable-looking parameters.

Every strategy parameter should be: named, documented, configurable, testable.

## Strategy state

Use the Phase 3 `StrategyStateStore`.

Possible runtime state includes:

- current signal
- last signal
- last signal timestamp
- last entry reference
- last exit reference
- trade count
- cooldown state
- strategy-specific state

Do not use Redis directly inside strategy logic. Use: `Strategy -> StrategyStateStore`.

## Signal model

Create a formal `Signal` model.

A signal should contain enough information for later risk and execution layers.

Potential fields:

- signal_id
- strategy_name
- timestamp
- underlying
- instrument_token
- tradingsymbol
- option_type
- expiry
- strike
- direction
- timeframe
- price
- signal_reason
- indicator_snapshot
- metadata

The exact fields should be determined from the existing architecture.

Do not include broker order IDs because no order exists yet.

Do not invent execution information.

## Signal direction

Support an explicit signal direction representation. For example: `BUY`, `SELL`, `HOLD`, `NO_SIGNAL`.

However, be careful: a strategy signal is NOT an executed order. A `BUY` signal means the strategy has identified a potential long opportunity. It does not mean the system should immediately place an order. Phase 8 and Phase 9 will determine whether and how the signal can become an actual trade.

## Signal ID

Every generated signal must have a unique deterministic identifier where possible.

The signal ID should allow duplicate detection.

Potential components: strategy, instrument, timeframe, signal timestamp, signal sequence/event identifier.

Do not use random IDs alone if deterministic identity is important.

Use the Phase 3 duplicate-event protection mechanism where appropriate.

## Duplicate signal protection

Before accepting a signal: check whether the signal has already been processed/generated.

Use the existing Redis deduplication infrastructure.

Expected behavior: new signal -> accepted; same signal -> duplicate/rejected.

Do not generate repeated identical signals indefinitely for the same event/candle unless explicitly allowed by the strategy.

## Strategy evaluation timing

Do not evaluate the strategy blindly on every tick unless there is a real requirement.

Prefer deterministic evaluation points such as a completed candle or another explicitly defined strategy event.

The strategy must clearly document whether it uses forming candles or completed candles.

Avoid look-ahead bias. A strategy must never use future candle information.

## No look-ahead bias

This is critical.

The strategy must only use information that would actually have been available at the decision timestamp.

Do not use: future candles, future prices, future indicator values, future option-chain information, future market state.

Tests must verify this where practical.

## Strategy logic

Implement the strategy defined by the project requirements.

IMPORTANT: Do not invent a strategy merely to make the implementation appear complete.

If the project already contains a defined strategy specification, implement that specification exactly.

If strategy requirements are incomplete or ambiguous: STOP and clearly report the missing strategy requirements before implementing arbitrary trading rules.

Do not silently invent: entry conditions, indicator thresholds, exit conditions, option selection rules, profit targets, stop losses.

Those must come from the project specification.

If the strategy specification is already present in the repository, use that as the authoritative source.

## Indicator consumption

Use Phase 6 `TechnicalIndicatorEngine`.

Do not recalculate indicators independently inside the strategy.

Example architecture: `MarketDataService -> IndicatorEngine -> Strategy`.

Do not duplicate: RSI calculations, EMA calculations, MACD calculations, ATR calculations, VWAP calculations.

If the strategy needs an indicator that Phase 6 does not provide, identify it and implement only the required extension in the appropriate indicator layer.

## Multi-timeframe support

If the strategy requires multiple timeframes: keep timeframe handling explicit. For example: primary timeframe, confirmation timeframe.

Do not mix candle timestamps incorrectly.

Ensure indicators are calculated against the correct timeframe.

Do not introduce multi-timeframe complexity unless required by the strategy specification.

## Option contract selection

Phase 5 provides option-chain discovery.

Phase 7 may consume option-chain information only if the strategy specification requires it.

If the strategy requires selecting an option contract: implement the strategy's contract-selection rules.

Do not hardcode a specific contract. Do not use arbitrary strike selection. Do not assume a fixed expiry.

Use Phase 5 `OptionChainService`.

Keep contract selection separate from signal generation where practical.

## Signal validation

Before returning a signal, validate: instrument exists, instrument is active, timestamp is valid, required market data exists, required indicators exist, strategy state is valid, signal ID is valid, required option contract exists if applicable.

Do not return malformed signals.

## Signal persistence

Determine whether signals should be persisted in PostgreSQL.

If signals are durable business events in the existing architecture, create/use the appropriate PostgreSQL repository.

A possible model may include: signal_id, strategy_name, timestamp, instrument_token, direction, price, reason, metadata, status, created_at.

Do not store unnecessary high-frequency duplicate signals.

If Redis is used for runtime signal state, PostgreSQL should remain the durable record where appropriate.

## Redis strategy state

Use Redis for: current strategy state, cooldown, last processed event, duplicate signal protection, temporary runtime state.

Use TTL where appropriate.

Do not store the entire strategy history in Redis. Do not make Redis the permanent signal database.

## Signal lifecycle

Design the signal lifecycle clearly. For example: `GENERATED -> VALIDATED -> ACCEPTED -> RISK_CHECK_PENDING`.

Do not implement the risk check yet. Do not implement execution.

The final state should make it clear that Phase 8 owns risk validation.

## Strategy cooldown

If the strategy specification contains cooldown behavior: implement it using `StrategyStateStore`.

Do not invent arbitrary cooldown durations.

Cooldown state must be deterministic and testable.

Do not use a cooldown as a substitute for proper duplicate-signal protection.

## Concurrency

The strategy engine may eventually run concurrently.

Use Phase 3 distributed-safety primitives where necessary.

Potential flow: strategy evaluation -> acquire strategy/event lock -> read state -> evaluate -> update state -> release lock.

Do not introduce unnecessary locking. Avoid GET followed by SET when an atomic operation is required.

## Error handling

Handle: missing market data, stale market data, missing indicators, invalid strategy state, invalid instrument, missing option contract, database failure, Redis failure, calculation failure.

Do not silently convert errors into BUY/SELL signals. A failure must never accidentally generate a trade signal.

## Market data safety

A strategy must not generate a signal from stale or unavailable market data.

Use Phase 4/6 market-data health/staleness information.

Define this behavior explicitly. Do not silently use an old price as if it were current.

## Logging

Use the existing logger.

Important events may include: `STRATEGY_STARTED`, `STRATEGY_EVALUATED`, `SIGNAL_GENERATED`, `SIGNAL_REJECTED`, `DUPLICATE_SIGNAL`, `STRATEGY_STATE_UPDATED`, `STRATEGY_ERROR`.

Do NOT log every tick. Do NOT log huge indicator payloads unnecessarily. Do not log credentials or secrets.

## Performance

The strategy engine may eventually operate during live market conditions.

Keep evaluation lightweight. Avoid: full database scans, full Redis scans, unnecessary API calls, recalculating all historical indicators, large state payloads, blocking operations.

Use existing Phase 6 data services.

## Testing — strategy

Create deterministic unit tests. Test: valid market context, missing market data, stale market data, missing indicators, valid strategy condition, invalid strategy condition, signal generation, no-signal condition, duplicate signal, strategy state update, cooldown if applicable, invalid instrument, missing option contract if applicable.

## Signal testing

Test that generated signals contain: unique signal ID, correct strategy name, correct timestamp, correct instrument, correct direction, correct price, correct reason, correct indicator snapshot where applicable.

Do not test order placement because order execution is not part of Phase 7.

## Look-ahead testing

Create tests demonstrating that the strategy does not use future information. For example: evaluate strategy at timestamp T; ensure data after T cannot affect the result.

## Replay / determinism

The same historical market context should produce the same strategy decision.

For identical market data, indicators, strategy configuration, and strategy state, the strategy should produce a deterministic result.

Avoid hidden randomness.

## Integration testing

Create integration tests for: `MarketDataService -> IndicatorEngine -> Strategy -> Signal -> StrategyStateStore`.

If signals are persisted: `Strategy -> SignalRepository -> PostgreSQL`.

Use isolated test databases and Redis. Do not use production services.

## Backward compatibility

All previous tests must continue to pass: Phase 1, Phase 2, Phase 3, Phase 4, Phase 5, Phase 6.

Do not break: Kite integration, WebSocket, Redis, PostgreSQL, instrument discovery, option-chain discovery, market-data engine, indicator engine.

## README update

Update README with: Phase 7 architecture, strategy input, indicator input, strategy state, signal model, signal lifecycle, duplicate signal protection, strategy configuration, option contract interaction if applicable.

Clear statement that: Phase 7 generates signals only. It does NOT place orders. It does NOT manage risk. It does NOT execute trades.

## Security and safety

Before completing Phase 7 verify: no credentials are hardcoded, no credentials are logged, no production services are used in tests, no unsafe deserialization, no accidental order placement, no Kite order API calls, no live trading, no hidden background trading process.

The application must remain safe to run in development.

## Critical safety requirement

Search the entire repository for any existing order-placement or trading-execution code before completing Phase 7.

If order-placement functionality already exists from an earlier phase unexpectedly: DO NOT invoke it. DO NOT connect the strategy engine to it. Do not enable it. Report it clearly in the final report.

Phase 7 must terminate at: SIGNAL GENERATED, not: ORDER PLACED.

## Definition of done

Phase 7 is complete only when:

- Strategy abstraction implemented.
- Strategy configuration implemented.
- Market context implemented.
- Strategy state integrated.
- Signal model implemented.
- Signal ID implemented.
- Duplicate signal protection implemented.
- Strategy evaluation timing defined.
- Look-ahead protection implemented.
- Strategy logic implemented ONLY according to the existing project strategy specification.
- Technical indicators consumed through Phase 6 infrastructure.
- Option-chain information consumed through Phase 5 where required.
- Signal validation implemented.
- Signal persistence implemented where appropriate.
- Redis runtime strategy state implemented where appropriate.
- Concurrency protection implemented where required.
- Error handling implemented.
- Logging implemented.
- Unit tests implemented.
- Look-ahead tests implemented.
- Deterministic strategy tests implemented.
- Integration tests implemented where practical.
- Phase 1 tests pass.
- Phase 2 tests pass.
- Phase 3 tests pass.
- Phase 4 tests pass.
- Phase 5 tests pass.
- Phase 6 tests pass.
- Phase 7 tests pass.
- README updated.
- NO orders placed.
- NO order execution implemented.
- NO risk management implemented.
- NO live trading enabled.
- NO Phase 8+ functionality implemented.

## Final report

At the end of implementation, report: Phase 1–6 architecture reviewed, strategy architecture implemented, files created, files modified, strategy configuration, market context, indicators consumed, strategy state design, signal model, signal ID design, duplicate-signal protection, option-chain interaction, signal persistence, Redis usage, PostgreSQL usage, concurrency approach, look-ahead protection, tests created, test results, Phase 1 test results, Phase 2 test results, Phase 3 test results, Phase 4 test results, Phase 5 test results, Phase 6 test results, Phase 7 test results, architectural decisions, known limitations, any missing strategy requirements, TODOs for Phase 8.

IMPORTANT: STOP AFTER PHASE 7.

Do not implement Phase 8. Do not implement risk management. Do not implement position sizing. Do not implement stop loss. Do not implement targets. Do not implement order placement. Do not place orders. Do not enable live trading. Do not implement broker reconciliation. Do not implement backtesting. Do not implement paper trading. Do not implement Phase 9 or later functionality.

---

## Addendum: Phase 7A — Actual Trading Strategy Specification and Implementation

### Important project context

Phase 1 through Phase 6 have been completed.

Phase 7 has also been run using the previous Phase 7 prompt.

However, Phase 7 currently contains a PLACEHOLDER strategy/signal decision implementation because the actual trading strategy conditions were not yet defined.

Now perform a controlled Phase 7A implementation.

The objective is to replace the placeholder strategy logic with the actual, explicitly defined trading strategy.

IMPORTANT:

- Do NOT invent a trading strategy silently.
- Do NOT randomly choose indicator thresholds.
- Do NOT assume that RSI, EMA, MACD, VWAP, or any other indicator should be used unless the strategy specification requires it.
- Do NOT optimize parameters for profitability.
- Do NOT claim that any configuration is profitable.
- Do NOT implement order execution.
- Do NOT implement risk management.
- Do NOT implement live trading.
- Do NOT implement Phase 8 or later phases.

### Inspect the existing implementation

Before making changes:

- Inspect the complete repository.
- Review Phase 1.
- Review Phase 2.
- Review Phase 3.
- Review Phase 4.
- Review Phase 5.
- Review Phase 6.
- Review the Phase 7 implementation that was just completed.
- Identify: strategy interface, strategy implementation, placeholder signal logic, `MarketContext`, indicator engine, strategy state store, signal model, signal repository, duplicate signal protection, option-chain integration, logging, configuration, tests, README.

Do not rewrite working infrastructure.

Reuse the existing Phase 7 architecture.

The goal is to replace ONLY the placeholder strategy decision logic where possible.

### First determine whether a real strategy specification already exists

Search the repository for: strategy rules, entry conditions, exit conditions, indicator conditions, signal conditions, NIFTY rules, BANK NIFTY rules, CE/PE rules, timeframe rules, confirmation rules, trading session rules, strategy documentation, TODO strategy notes, configuration values, comments describing intended strategy behavior.

If an explicit strategy specification already exists in the repository: use that specification as the authoritative source. Do not invent alternative rules.

If no complete strategy specification exists:

- DO NOT IMPLEMENT ARBITRARY STRATEGY LOGIC.
- Instead, produce a STRATEGY SPECIFICATION PROPOSAL for review.
- The proposal must be presented before modifying the placeholder logic.

### Strategy specification must define the following

The actual strategy must explicitly define:

#### A. Instruments

Which underlying instruments are supported? For example: NIFTY, BANK NIFTY. Do not assume additional instruments.

#### B. Trading session

Define: allowed trading start time, allowed trading end time, whether signals are allowed near market close, whether the strategy operates throughout the session or only during specific windows.

#### C. Primary timeframe

Define the main candle timeframe. Examples: 1 minute, 5 minute, 15 minute, etc. Do not choose a timeframe arbitrarily.

#### D. Confirmation timeframe

If required, define a second timeframe. If not required, explicitly state that the strategy is single-timeframe.

#### E. Indicators

List every indicator used. For each indicator specify: name, period, source price, timeframe, required historical observations.

Example structure:

```
EMA
period = X
source = close
timeframe = Y
```

Do not invent X or Y.

#### F. Entry conditions

Define exact mathematical conditions.

Example structure: Condition 1, Condition 2, Condition 3.

Then define whether the relationship is: AND, OR, conditional.

Do not use vague descriptions such as "strong trend", "good momentum", "bullish market", "high volume" unless these are converted into measurable rules.

#### G. Confirmation

Define exactly what confirms an entry. Examples may include: candle close, indicator crossover, volume condition, price relationship, multi-timeframe confirmation.

But only use conditions explicitly specified by the strategy.

#### H. CE/PE selection

If the strategy trades options, define: how CALL vs PUT is selected, how expiry is selected, how strike is selected, whether ATM is used, whether ITM/OTM is used, how many strikes away, how strike distance is determined.

Do not hardcode a particular option contract. Use Phase 5 `OptionChainService`.

#### I. Signal conditions

Define exactly when the engine produces: `BUY`, `SELL`, `NO_SIGNAL`.

If `HOLD` is supported, define its meaning.

Remember: Signal != Order.

#### J. Signal invalidation

Define when an otherwise valid setup becomes invalid.

#### K. Duplicate signals

Define whether the same setup can generate another signal. Use the existing Phase 3 deduplication infrastructure.

#### L. Cooldown

If cooldown is required, define its exact behavior. Do not invent a cooldown period.

### If strategy is not defined, stop before code changes

If no complete strategy exists in the repository:

- Do NOT select indicator conditions yourself.
- Do NOT implement a random strategy.
- Do NOT modify the placeholder signal behavior.
- Instead, create a file/document such as `docs/strategy_specification.md` containing a clearly structured proposed specification template.
- Then report: STRATEGY SPECIFICATION REQUIRED, and list exactly what information is missing.
- Do not proceed into arbitrary implementation. This is intentional.

### If strategy is defined, implement it

If a complete strategy specification exists:

- Replace the Phase 7 placeholder strategy logic with the specified rules.
- Use the existing: `MarketContext`, `IndicatorEngine`, `OptionChainService`, `StrategyStateStore`, `Signal` model, signal repository, Redis deduplication.
- Do not duplicate existing services.

### Strategy engine

The strategy should have a clean structure. Conceptually:

```
MarketContext -> Indicator Values -> Condition Evaluation -> Strategy Decision -> Signal Validation -> Signal
```

Keep individual conditions modular where practical. Avoid one enormous function containing every condition.

### Condition evaluation

Conditions must be explicit and testable. For example: `indicator_a > threshold`, `indicator_a crosses indicator_b`, `price > moving_average`, `candle_close > previous_high`.

Only use conditions actually defined by the strategy.

Do not hide trading logic inside utility functions without documentation.

### Indicator dependencies

For every strategy condition: verify that the required indicator exists in Phase 6.

If an indicator is missing: do not duplicate its calculation inside the strategy. Extend Phase 6's indicator infrastructure only if necessary. Document the new dependency.

### Option selection

If the strategy trades options: use Phase 5's option-chain infrastructure. Do not query Kite directly from the strategy.

Expected architecture: `Strategy -> OptionChainService -> Instrument Repository -> PostgreSQL`.

The strategy may request the appropriate contract according to the explicit strategy rules. Do not place orders.

### Signal content

Every generated signal should contain the existing Phase 7 `Signal` model fields.

Ensure the signal includes enough information for Phase 8.

Potential information: signal_id, strategy_name, timestamp, underlying, instrument_token, tradingsymbol, option_type, expiry, strike, direction, timeframe, price, signal_reason, indicator_snapshot, metadata.

Do not include order IDs. No order exists yet.

### Signal reason

Every generated signal should contain a machine-readable and human-readable reason. For example: conditions satisfied, indicator confirmation, timeframe confirmation, option contract selected.

Do not generate vague reasons. The reason should allow debugging later.

### Indicator snapshot

When a signal is generated, capture the relevant indicator values used by the strategy.

Do not store unnecessary entire market history.

The snapshot should allow later debugging of: why was this signal generated, what were the indicator values, what was the market state, what strategy parameters were active.

### No look-ahead bias

This is mandatory. The strategy must only use information available at the signal timestamp.

Do not use: future candles, future indicator values, future prices, future volume, future option-chain information, future market state.

Test this explicitly.

### Signal timing

Follow the strategy's defined evaluation timing.

If the strategy uses completed candles: do not generate a signal before the candle is complete.

If the strategy uses forming candles: explicitly document that behavior.

Do not change the timing arbitrarily.

### Market data validation

Before strategy evaluation: verify market data is available, verify market data is not stale, verify required candles exist, verify required indicators exist.

If required data is missing: return `NO_SIGNAL` or an appropriate non-trading result. Never generate a BUY/SELL signal from incomplete required data.

### Strategy state

Use the existing `StrategyStateStore`.

Persist the state required by the strategy. Examples: last signal, last signal timestamp, last processed candle, cooldown, strategy-specific state.

Do not store unnecessary data.

### Duplicate signal protection

Use the existing Phase 3 duplicate-event protection.

The same strategy event must not repeatedly produce identical signals unless the strategy explicitly permits repeated signals.

Test: first evaluation -> accepted; same event -> duplicate/rejected; new valid event -> accepted.

### Concurrency

Use existing Phase 3 distributed locks only where required. Avoid unnecessary locking.

Where atomic state transitions are required, use appropriate atomic operations. Do not build a new locking system.

### Strategy parameters

All strategy parameters must be centralized. Do not scatter values throughout the code.

Parameters should be: configurable, documented, testable, versionable where appropriate.

Examples: indicator periods, thresholds, timeframes, confirmation requirements, cooldown.

Do not optimize these parameters in this phase.

### Strategy version

Add a strategy version identifier if appropriate. For example: `strategy_name`, `strategy_version`.

This will allow future signals to be traced to the exact strategy configuration that generated them.

Do not create complex version-management infrastructure unnecessarily.

### Logging

Use the existing logger.

Log important events such as: `STRATEGY_EVALUATED`, `STRATEGY_CONDITION_FAILED`, `SIGNAL_GENERATED`, `SIGNAL_REJECTED`, `DUPLICATE_SIGNAL`, `STRATEGY_STATE_UPDATED`, `STRATEGY_ERROR`.

Do not log every tick. Do not log unnecessary large indicator payloads. Do not log credentials.

### Testing — every condition

Create deterministic unit tests for every strategy condition.

For each condition test: condition true, condition false, boundary value, missing data, invalid data.

Ensure AND/OR logic is tested explicitly.

### Entry signal tests

Test complete scenarios: all entry conditions satisfied, one required condition missing, multiple conditions missing, confirmation missing, stale market data, insufficient candles, missing indicator, invalid option contract, duplicate signal, new valid signal.

### Option selection tests

If options are part of the strategy, test: correct expiry selection, correct strike selection, correct CE selection, correct PE selection, missing contract, invalid contract, expired contract, multiple matching contracts.

Do not test order placement.

### Determinism testing

The same market data, indicator values, strategy configuration, and strategy state must produce the same strategy decision.

Do not use randomness.

### Historical replay compatibility

The strategy should be structured so that later Phase 11 backtesting can feed historical candles into the same strategy engine.

Do not build the complete backtesting engine now. Do not implement paper trading now.

Only ensure the strategy interface can consume historical market context without depending directly on live WebSocket objects.

### Phase 8 boundary

The strategy produces: SIGNAL.

Phase 8 will later determine: whether the signal passes risk controls, position size, maximum exposure, stop loss, target, trade limits.

Do not implement these in Phase 7A.

### Phase 9 boundary

Phase 9 will later determine how a risk-approved signal becomes an actual broker order.

Do not implement: order placement, order modification, order cancellation, execution tracking, in Phase 7A.

### Security

Verify: no Kite credentials are hardcoded, no credentials are logged, no orders can be placed by the strategy engine, no accidental Kite order API calls exist in the Phase 7 execution path, development execution remains safe.

### README update

Update README with: actual strategy name, strategy version, supported instruments, timeframes, indicators, indicator periods, entry conditions, confirmation conditions, option-selection rules, signal lifecycle, duplicate-signal protection, strategy state.

Clear statement: Phase 7A generates signals only. Phase 8 handles risk. Phase 9 handles order execution.

### Test regression

Run all tests: Phase 1 tests, Phase 2 tests, Phase 3 tests, Phase 4 tests, Phase 5 tests, Phase 6 tests, Phase 7 tests, Phase 7A tests.

Fix regressions without rewriting unrelated components.

### Definition of done

Phase 7A is complete only when:

- The actual strategy specification is explicitly defined.
- Every indicator used by the strategy is documented.
- Every indicator period is documented.
- Every entry condition is explicit.
- Every confirmation condition is explicit.
- Option-selection rules are explicit if options are traded.
- Signal timing is explicit.
- Duplicate-signal behavior is explicit.
- Strategy state behavior is explicit.
- Placeholder strategy logic is replaced ONLY if a complete strategy specification exists.
- Strategy implementation is modular.
- Strategy is deterministic.
- No look-ahead bias exists.
- Market-data staleness is handled.
- Signals are validated.
- Signals have unique identifiers.
- Signal deduplication works.
- Relevant indicator values are captured.
- Strategy tests cover every condition.
- Integration tests pass.
- All previous phase tests pass.
- README is updated.
- No risk management is implemented.
- No order execution is implemented.
- No live trading is enabled.

### Final report

At the end, report: whether a complete strategy specification was found.

If not found: what information is missing, where the strategy specification template was created, confirmation that placeholder logic was NOT replaced with invented rules.

If found: strategy name, strategy version, supported instruments, timeframes, indicators, indicator periods, entry conditions, confirmation conditions, option-selection rules, signal lifecycle, duplicate protection, strategy-state design.

Also report: files created, files modified, tests created, test results, regression test results for Phases 1–7, known limitations, TODOs for Phase 8.

IMPORTANT: STOP AFTER PHASE 7A.

Do not implement Phase 8. Do not implement risk management. Do not implement position sizing. Do not implement stop loss. Do not implement targets. Do not implement order placement. Do not place orders. Do not enable live trading. Do not implement broker reconciliation. Do not implement backtesting. Do not implement paper trading. Do not implement Phase 9 or later functionality.
