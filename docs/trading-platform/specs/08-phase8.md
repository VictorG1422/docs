# Phase 8 — Risk & Position Management

## Project roadmap context

This trading system is being built in controlled phases.

### Completed

- **Phase 1 — Core Application Foundation.** Core application architecture, configuration, logging, health checks, project structure, and foundational services.
- **Phase 2 — PostgreSQL Database.** PostgreSQL connectivity, durable persistence, models/repositories/migrations as required. PostgreSQL is the durable source of truth.
- **Phase 3 — Redis State & Cache.** Redis configuration, connection pooling, health checks, key builder, state management, TTL, market state, strategy state, position state, distributed locks, duplicate-event protection, caching, serialization, and tests. Redis is runtime/fast state, NOT the permanent source of truth.
- **Phase 4 — Kite Connect & Market Data Foundation.** Kite authentication/client infrastructure, instrument synchronization, WebSocket, subscription management, tick processing, Redis market state, reconnection, stale-data detection, and tests.
- **Phase 5 — Instrument & Option-Chain Foundation.** Underlying/expiry/strike discovery, CE/PE contracts, contract validation, option-chain representation, repositories, caching, and tests.
- **Phase 6 — Market Data & Technical Analysis.** Tick validation, candle aggregation, historical market-data interfaces, OHLCV data, technical indicators, market-data quality validation, caching, and tests.
- **Phase 7 — Trading Strategy & Signal Engine.** Strategy framework, market context, indicator consumption, strategy state, signal model, signal IDs, duplicate protection, deterministic evaluation, and signal generation.
- **Phase 7A — Actual Strategy Specification.** The actual strategy conditions, indicators, timeframes, confirmation rules, and option-selection rules have been defined and implemented where specified.

### This phase

**Phase 8 — Risk & Position Management. THIS PHASE.**

### Remaining roadmap

- **Phase 9 — Order Execution.** Will later convert risk-approved trade intents into actual broker orders.
- **Phase 10 — Broker Reconciliation.**
- **Phase 11 — Backtesting & Paper Trading.**
- **Phase 12 — Production & Live-Trading Readiness.**
- **Phase 13 — Optimization.**

Do NOT implement any later phase.

## Phase 8 objective

Implement a deterministic risk-management and position-management layer that receives signals from Phase 7 and determines whether those signals are permitted to proceed toward execution.

Expected flow:

```
Phase 7 Strategy
      ↓
   Signal
      ↓
Phase 8 Risk Engine
      ↓
Risk Validation
      ↓
Position Sizing
      ↓
Stop-Loss / Target Calculation
      ↓
Trade Intent
      ↓
Phase 9 Order Execution
```

IMPORTANT:

- Phase 8 MUST NOT place any broker orders.
- Phase 8 MUST NOT call Kite order-placement APIs.
- Phase 8 produces a risk-approved or rejected TRADE INTENT.
- Phase 9 will consume the trade intent.

## Inspect existing project first

Before making changes:

- Inspect the complete repository.
- Review: Phase 1, Phase 2, Phase 3, Phase 4, Phase 5, Phase 6, Phase 7, Phase 7A.
- Review: configuration, logging, PostgreSQL, Redis, Kite client, market data, candles, indicators, option-chain service, strategy engine, signal model, strategy state, position state, repositories, distributed locks, tests, README.
- Search for any existing: order placement, order execution, position sizing, risk logic, stop-loss, target, broker reconciliation.
- Do not duplicate existing functionality.

Before implementation, briefly explain:

1. Current architecture.
2. How Phase 8 receives Phase 7 signals.
3. Risk-engine architecture.
4. Position-management architecture.
5. Position-sizing architecture.
6. Stop-loss/target architecture.
7. PostgreSQL responsibilities.
8. Redis responsibilities.
9. Phase 9 interface.
10. Files to create.
11. Files to modify.

Then implement Phase 8.

## Critical strategy/risk separation

Phase 7 answers: "Is there a strategy setup?"

Phase 8 answers: "Is this setup allowed to become a trade, and if so, what risk-controlled trade intent should be produced?"

Phase 8 MUST NOT change the strategy's signal logic.

Do not modify strategy entry conditions to make them pass risk checks.

Do not generate new BUY/SELL signals in Phase 8.

## Risk configuration

Do NOT hardcode risk parameters.

Create centralized configuration.

Potential configuration includes: maximum capital allocation, maximum risk per trade, maximum number of open positions, maximum daily loss, maximum daily trades, maximum exposure, maximum position value, maximum quantity, maximum premium exposure, maximum loss, maximum consecutive losses, cooldown, allowed trading hours, minimum liquidity requirements, maximum spread, maximum stale-data age.

IMPORTANT: Only implement limits that are actually defined by the project requirements. Do NOT invent arbitrary percentages or rupee values. If critical risk parameters are not defined: identify them clearly. Do not silently invent financial risk limits.

## Risk policy

Create a structured `RiskPolicy` abstraction.

Conceptually: `RiskPolicy -> RiskEngine`.

The policy should contain configurable limits. Example conceptual structure:

```
RiskPolicy
  max_risk_per_trade
  max_daily_loss
  max_open_positions
  max_daily_trades
  max_exposure
  max_quantity
  trading_window
  data_staleness_limit
```

Use the project's actual configuration conventions. Do not create a second configuration system.

## Risk decision model

Create a structured result. For example: `RiskDecision`.

Possible outcomes: `APPROVED`, `REJECTED`.

The decision should contain: approved, reason, risk_checks, timestamp, signal_id, metadata.

If rejected, provide a clear machine-readable reason. Examples: `MAX_DAILY_LOSS_EXCEEDED`, `MAX_OPEN_POSITIONS`, `MAX_EXPOSURE`, `INVALID_SIGNAL`, `STALE_MARKET_DATA`, `INVALID_PRICE`, `INVALID_QUANTITY`, `OUTSIDE_TRADING_WINDOW`, `INSUFFICIENT_CAPITAL`, `DUPLICATE_SIGNAL`.

Do not use vague rejection reasons.

## Risk check pipeline

Create a deterministic sequence of risk checks. Conceptually:

```
Signal
  ↓
Signal validation
  ↓
Market-data validation
  ↓
Trading-window validation
  ↓
Existing-position validation
  ↓
Daily-loss validation
  ↓
Exposure validation
  ↓
Position-size calculation
  ↓
Quantity validation
  ↓
Stop-loss validation
  ↓
Target validation
  ↓
Risk decision
  ↓
Trade Intent
```

## Capital and account state

Create an abstraction for account/risk state.

Do NOT assume Redis is authoritative for broker account balance.

Potential information: available capital, used capital, available margin, current exposure, realized P&L, unrealized P&L, daily P&L, open positions, daily trade count.

The exact source must be documented.

Since actual broker reconciliation belongs to Phase 10, do not pretend broker state is always synchronized.

If broker account information is unavailable: do not fabricate it. Return an explicit unavailable state where necessary.

## Position sizing

Implement a reusable `PositionSizer`.

Conceptually: `Signal + RiskPolicy + AccountState + EntryPrice + StopLoss -> PositionSizer -> Quantity`.

The position size must respect: risk-per-trade, available capital, maximum exposure, maximum quantity, instrument lot size, option contract constraints, other configured limits.

## Financial precision

Use appropriate exact numeric types.

Do NOT casually use binary floating point for: prices, premium, capital, risk, P&L, quantity calculations where exact arithmetic matters.

Use `Decimal`/`Numeric` conventions consistent with the existing project.

Document rounding behavior.

## Lot size

For derivatives/options, quantity must respect the instrument's lot size.

Example: calculated quantity = 137, lot size = 75, valid quantity must be rounded according to the defined policy.

Do NOT invent lot sizes. Use Phase 5 instrument metadata where available. Do not hardcode NIFTY/BANK NIFTY lot sizes.

## Position limits

Implement configurable checks for: maximum open positions, maximum quantity, maximum exposure, maximum capital allocation, maximum risk, maximum contracts.

Do not create limits that conflict with existing configuration.

## Existing position check

Before approving a new trade intent: check current position state.

Use existing Phase 3 `PositionStateStore` where appropriate.

Also recognize: Redis is runtime state; PostgreSQL is durable state; broker state may later become authoritative through Phase 10 reconciliation.

Do not assume Redis alone proves the actual broker position.

## Duplicate position protection

Prevent accidental repeated trade intents for the same signal/setup.

Use: signal_id, strategy state, position state, existing Phase 3 deduplication infrastructure.

Do not rely solely on an application-level boolean.

## Daily loss limit

If the project specifies a daily loss limit: implement it as a hard risk gate.

Conceptually: `daily P&L >= configured loss threshold -> reject new trade intents`.

Define precisely whether the limit is based on: realized P&L, realized + unrealized P&L, or another explicitly defined metric.

Do not invent this policy.

## Daily trade limit

If configured: track daily trade count. Reject new trade intents after the configured maximum.

Ensure restart behavior is deterministic.

Use PostgreSQL for durable records where appropriate. Use Redis only for runtime acceleration/state.

## Exposure limit

Calculate current exposure plus proposed exposure.

Example: `existing exposure + new trade exposure <= maximum exposure`.

Use appropriate financial precision.

Do not count a signal as exposure until the Phase 9 execution actually occurs.

Clearly distinguish: proposed exposure from actual exposure.

## Stop loss

If the strategy specification defines stop-loss behavior: implement the stop-loss calculation.

If stop-loss is NOT defined: do not invent one. Instead report that a required risk parameter is missing.

Stop-loss calculation must be deterministic.

Do not place or modify stop-loss orders. Phase 9/10 will handle broker-side execution and reconciliation.

## Target

If the project defines a target: implement the calculation.

If no target is defined: do not invent one. Do not create arbitrary risk/reward ratios. Do not place target orders.

## Risk/reward

If the strategy/risk specification requires a minimum risk/reward ratio: calculate it explicitly.

Example: `reward / risk`.

Reject trades that fail the configured requirement.

Do not invent the threshold.

## Option-specific risk

If trading options, consider configured controls for: premium, contract lot size, quantity, notional exposure, maximum premium exposure, liquidity, bid/ask spread, contract validity, expiry, option type, strike, instrument status.

Do not invent liquidity thresholds. Use Phase 5 option metadata.

Do not query Kite directly from the risk engine if an existing service abstraction can provide the required information.

## Liquidity check

If required by the project: validate that the option is sufficiently liquid.

Potential data: bid, ask, spread, volume, open interest.

Do not fabricate missing values. If required information is unavailable: return a clear risk-check result.

## Market-data staleness

Risk decisions must not rely on stale market data.

Use Phase 4/6 staleness infrastructure.

If the current price is stale beyond the configured threshold: reject the trade intent.

Do not silently substitute an old price.

## Trading window

If risk policy defines trading hours: reject trade intents outside the allowed window.

Use timezone-aware timestamps. Do not hardcode timezone assumptions. Use the application's existing timezone configuration.

## Position state model

Create or extend the position model as required.

Potential fields: instrument_token, tradingsymbol, underlying, option_type, expiry, strike, quantity, average_entry_price, current_price, position_status, stop_loss, target, strategy, signal_id, opened_at, updated_at, realized_pnl, unrealized_pnl.

Do not duplicate the Phase 3 position abstraction unnecessarily.

## Position lifecycle

Define clear states. For example: `NONE`, `PENDING`, `OPEN`, `CLOSING`, `CLOSED`.

IMPORTANT: Phase 8 can create a trade intent and expected position state. Phase 9 will later create actual broker execution.

Do not mark a position as `OPEN` merely because a signal was approved. An approved intent is NOT an executed position.

## Trade intent model

Create a `TradeIntent` model.

Potential fields: intent_id, signal_id, strategy_name, timestamp, instrument_token, tradingsymbol, underlying, option_type, expiry, strike, direction, quantity, entry_price_reference, stop_loss, target, risk_amount, maximum_loss, capital_required, reason, risk_decision, strategy_version, metadata.

The exact fields should follow the existing architecture.

This object is the main output of Phase 8. It will later be consumed by Phase 9.

## Trade intent status

Define clear statuses such as: `RISK_APPROVED`, `RISK_REJECTED`, `PENDING_EXECUTION`.

Do not mark it: `EXECUTED`, `FILLED`, `COMPLETED` unless a future execution layer provides that information.

## Persistence

Persist durable risk decisions/trade intents in PostgreSQL where appropriate.

Potential records: risk decision, trade intent, risk rejection reason, position-sizing calculation, strategy version, risk policy version.

Do not persist every intermediate calculation unnecessarily.

## Redis usage

Use Redis for fast runtime state such as: current risk state, daily counters, cooldowns, temporary locks, current position cache, deduplication.

Do not make Redis the durable source of truth. Where a state is recovery-critical, PostgreSQL should remain authoritative where appropriate.

## Atomic daily counters

If Redis is used for: daily trade count, daily risk counters, cooldowns, similar counters, use atomic operations where race conditions are possible.

Do not implement `GET` then `SET` when an atomic increment/set-if-not-exists operation is required.

## Concurrency

Risk evaluation can potentially happen concurrently.

Use existing Redis distributed-lock infrastructure where necessary.

Potential flow:

```
Signal
  ↓
Acquire risk lock
  ↓
Read account/position/risk state
  ↓
Run risk checks
  ↓
Calculate position size
  ↓
Persist risk decision/trade intent
  ↓
Update runtime state
  ↓
Release lock
```

Avoid unnecessary global locks. Prefer narrow, instrument/strategy/signal-specific locking.

## Fail-safe behavior

Risk failures must NOT accidentally become approvals.

Examples: missing account state, missing price, invalid quantity, stale market data, database failure, critical Redis failure, unknown position state should NOT silently produce `RISK_APPROVED`.

## Redis failure

If Redis is unavailable: determine whether the affected risk check is critical.

For critical state: do not silently approve. Return a clear risk-system-unavailable result.

Do not invent fail-open behavior. Do not invent fail-closed policy where the project's requirements have not defined it. Document the decision.

## Database failure

If PostgreSQL is unavailable while durable risk records are required:

- Do not claim that a risk decision was durably recorded.
- Do not silently discard the event.
- Return a clear error state.
- Do not place orders.

## Error handling

Handle: invalid signal, missing market data, stale market data, missing account state, invalid position, invalid price, invalid quantity, invalid lot size, missing option metadata, Redis failure, PostgreSQL failure, calculation failure, serialization failure, configuration errors.

Do not silently swallow risk failures.

## Logging

Use the existing logger.

Important events: `RISK_EVALUATION_STARTED`, `RISK_CHECK_FAILED`, `RISK_APPROVED`, `RISK_REJECTED`, `POSITION_SIZE_CALCULATED`, `TRADE_INTENT_CREATED`, `RISK_ERROR`, `POSITION_STATE_UPDATED`.

Do not log: credentials, secrets, unnecessary full account information, sensitive configuration, huge market-data payloads.

## Auditability

Risk decisions should be explainable.

For every rejected trade intent, record: signal_id, timestamp, risk check, result, reason, relevant safe numeric values, risk policy version if available.

For approved intents, record: approved checks, quantity, risk amount, capital requirement, stop loss if applicable, target if applicable, strategy version, risk policy version.

This is important for later debugging and audit.

## Risk policy version

Add a risk-policy version. For example: `risk_policy_version`.

This should allow later analysis of: which risk configuration approved/rejected this trade intent?

## Testing — risk engine

Create deterministic unit tests for: valid signal, invalid signal, stale market data, invalid price, missing market data, maximum risk, maximum exposure, maximum position count, maximum quantity, daily loss limit, daily trade limit, trading window, insufficient capital, duplicate signal, duplicate position, invalid option contract, missing option metadata.

## Position-sizing tests

Test: risk-based sizing, capital-based sizing, maximum quantity, lot-size rounding, minimum valid quantity, quantity becoming zero, Decimal precision, extreme prices, extreme stop-loss distance.

## Stop-loss/target tests

If implemented, test: valid stop-loss, invalid stop-loss, valid target, invalid target, risk/reward, boundary values, missing configuration, precision.

## Trade-intent tests

Test that an approved intent contains: intent_id, signal_id, instrument, direction, quantity, entry reference, risk amount, capital requirement, stop-loss if applicable, target if applicable, strategy version, risk-policy version.

## Rejection tests

Every rejection must: not create an executable order, contain a deterministic reason, be auditable, not accidentally alter position state incorrectly.

## Concurrency testing

Test concurrent evaluation of the same signal.

Expected behavior: one evaluation accepted; duplicate/concurrent evaluation rejected or deduplicated. No duplicate trade intents should be created.

## Integration testing

Create integration tests covering:

```
Phase 7 Signal
  ↓
Phase 8 Risk Engine
  ↓
Risk Decision
  ↓
Position Sizing
  ↓
Trade Intent
  ↓
PostgreSQL/Redis
```

Use isolated test environments. Never use production Kite, PostgreSQL, Redis, account state, or credentials.

## Phase 9 interface

Define a clean interface for Phase 9.

Example: `Phase 8 -> TradeIntent -> Phase 9 ExecutionEngine`.

Phase 8 must not know how the broker order will actually be placed.

Do not call Kite order APIs. Do not create broker orders.

## Backward compatibility

All existing tests must pass: Phase 1, Phase 2, Phase 3, Phase 4, Phase 5, Phase 6, Phase 7, Phase 7A.

Do not break existing: market data, strategy, signals, Redis, PostgreSQL, Kite infrastructure, instrument discovery, option-chain discovery.

## README update

Update README with: Phase 8 architecture, risk engine, risk policy, position sizing, position state, trade intent, risk checks, daily limits, exposure limits, stop-loss/target responsibility, Redis vs PostgreSQL responsibilities, risk-policy versioning, Phase 9 boundary.

Clearly state: Phase 8 DOES NOT place orders. Phase 8 DOES NOT execute trades. Phase 8 produces `TradeIntent` only.

## Security

Verify: no credentials hardcoded, no credentials logged, no production services used by tests, no accidental Kite order API calls, no order-placement calls, no live trading, no unsafe deserialization, no secrets committed.

## Critical safety check

Search the repository for: `place_order`, `place_order_variety`, `modify_order`, `cancel_order`, KiteConnect order methods, any broker execution methods.

If any exist: DO NOT call them. DO NOT connect them to Phase 8. Report them in the final report.

Phase 8 must stop at: RISK-APPROVED TRADE INTENT, NOT: ORDER PLACED.

## Definition of done

Phase 8 is complete only when:

- `RiskPolicy` implemented.
- `RiskEngine` implemented.
- Risk checks implemented.
- Risk configuration centralized.
- `PositionSizer` implemented.
- Financial precision handled.
- Lot-size handling implemented.
- Position state integrated.
- Daily limits implemented where specified.
- Exposure limits implemented where specified.
- Capital checks implemented where specified.
- Market-data staleness checks implemented.
- Trading-window checks implemented where specified.
- Stop-loss implemented where specified.
- Target implemented where specified.
- Risk/reward implemented where specified.
- `TradeIntent` implemented.
- Risk decision implemented.
- Risk rejection reasons implemented.
- Redis runtime state integrated.
- PostgreSQL durable records implemented where appropriate.
- Concurrency protection implemented.
- Duplicate trade-intent protection implemented.
- Auditability implemented.
- Risk-policy versioning implemented.
- Unit tests implemented.
- Concurrency tests implemented.
- Integration tests implemented where practical.
- All Phase 1–7A tests pass.
- README updated.
- NO orders placed.
- NO order execution implemented.
- NO live trading enabled.
- NO broker reconciliation implemented.
- NO Phase 9+ functionality implemented.

## Final report

At the end, report: Phase 1–7A architecture reviewed, risk architecture, `RiskPolicy`, risk checks, position-sizing approach, financial precision approach, lot-size handling, position-state design, daily-limit handling, exposure-limit handling, stop-loss handling, target handling, `TradeIntent` model, risk rejection reasons, Redis usage, PostgreSQL usage, concurrency approach, duplicate protection, auditability, risk-policy versioning, files created, files modified, tests created, test results, Phase 1–7A regression results, missing/undefined risk parameters, architectural decisions, known limitations, TODOs for Phase 9.

IMPORTANT: STOP AFTER PHASE 8.

Do NOT implement Phase 9. Do NOT place orders. Do NOT call Kite order APIs. Do NOT implement order execution. Do NOT implement broker reconciliation. Do NOT implement backtesting. Do NOT implement paper trading. Do NOT implement production/live-trading deployment. Do NOT implement Phase 10 or later functionality.

Final output of Phase 8:

```
SIGNAL
  ↓
RISK ENGINE
  ↓
RISK DECISION
  ↓
TRADE INTENT
  ↓
STOP
```

Phase 9 starts from `TradeIntent`.
