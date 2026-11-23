# Phase 10 — Broker Reconciliation & State Consistency

## Project Roadmap

**Completed:**
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

**This phase:** Phase 10 — Broker Reconciliation & State Consistency

**Upcoming:**
- Phase 11 — Backtesting & Paper Trading
- Phase 12 — Production & Live-Trading Readiness
- Phase 13 — Optimization

**IMPORTANT:** Implement Phase 10 ONLY. Do NOT implement Phase 11 or later.

## Phase 10 Objective

The purpose of Phase 10 is to ensure that:

BROKER STATE matches APPLICATION STATE matches POSTGRESQL DURABLE STATE matches REDIS RUNTIME STATE where appropriate.

The broker is the authoritative source for actual broker-side:
- orders
- executions
- fills
- positions

PostgreSQL remains the application's durable historical record.
Redis remains fast runtime/cache state.

Phase 10 must detect and safely resolve differences between these systems.

## Core Architecture

```
Kite Broker
  ↓
Broker Reconciliation Engine
  ↓
Compare
  ┌───────────────┐
  │ PostgreSQL    │
  │ Redis         │
  │ Broker State  │
  └───────────────┘
  ↓
Differences
  ↓
Reconciliation Decision
  ↓
PostgreSQL Update
  ↓
Redis Runtime Update
  ↓
Consistent Application State
```

## Inspect the Complete Project First

Before making changes:
- Inspect the complete repository.
- Review Phases 1–9.
- Inspect: Kite authentication, Kite broker adapter, order execution engine, TradeIntent, order models, execution models, position models, PostgreSQL repositories, Redis state stores, distributed locks, deduplication, market data, configuration, logging, tests, README.
- Search for: order status, order history, positions, broker order ID, filled quantity, average price, execution state, reconciliation, position updates.
- Do not duplicate existing functionality.

Before implementation, briefly explain:
- Current order lifecycle.
- Current position lifecycle.
- Broker data available through Kite.
- PostgreSQL state.
- Redis state.
- Reconciliation architecture.
- Conflict resolution strategy.
- Restart recovery strategy.
- Files to create.
- Files to modify.

Then implement Phase 10.

## Critical Boundary

Phase 10 is a RECONCILIATION system. It is NOT:
- strategy
- signal generation
- risk engine
- order-generation engine
- position-sizing engine
- backtesting
- optimization

Phase 10 MUST NOT generate new trading signals.
Phase 10 MUST NOT modify strategy conditions.
Phase 10 MUST NOT calculate new trade opportunities.

## Order Reconciliation

Reconcile Kite broker orders against PostgreSQL order/execution records.

Compare: broker_order_id, intent_id where available, instrument, transaction type, quantity, filled quantity, pending quantity, order type, price, average fill price, status, timestamps.

Detect: missing broker order, missing database order, status mismatch, quantity mismatch, fill mismatch, price mismatch, unknown broker order, unknown application order.

## Position Reconciliation

Reconcile Kite broker positions against PostgreSQL position state.

Compare: instrument, tradingsymbol, quantity, average entry price, product, position status, realized P&L where available.

Detect: missing position, unexpected position, quantity mismatch, average-price mismatch, closed-vs-open mismatch, unexpected broker position, application position missing at broker.

## Broker Is Authoritative For Actual Execution

For actual broker-side reality: Broker state is authoritative.

Example: If PostgreSQL says quantity = 0 but broker says quantity = 75, the system MUST NOT assume PostgreSQL is correct. The discrepancy must be detected and handled according to the reconciliation policy.

## PostgreSQL Role

PostgreSQL remains the durable application record.

After reconciliation:
- Persist the broker-observed state and reconciliation result.
- Do not delete historical records merely because the broker state changed.
- Maintain audit history.

## Redis Role

Redis is runtime state.

After successful reconciliation:
- update Redis position state
- update relevant order state
- remove stale runtime state where appropriate

Do not treat Redis as the durable source of truth.

## Reconciliation Result Model

Create a structured `ReconciliationResult`.

Potential fields: reconciliation_id, timestamp, environment, scope, status, orders_checked, positions_checked, mismatches_detected, mismatches_resolved, mismatches_unresolved, errors, duration, metadata.

Possible status: CONSISTENT, RECONCILED, MISMATCH_DETECTED, RECONCILIATION_FAILED, PARTIALLY_RECONCILED.

Do not hide unresolved mismatches.

## Order Reconciliation Result

For each order: local state, broker state, comparison result, difference, resolution, timestamp.

Potential outcomes: MATCHED, BROKER_NEW, LOCAL_NEW, STATUS_MISMATCH, QUANTITY_MISMATCH, FILL_MISMATCH, PRICE_MISMATCH, UNKNOWN.

## Position Reconciliation Result

For each position: local state, broker state, comparison result, difference, resolution.

Potential outcomes: MATCHED, BROKER_ONLY, LOCAL_ONLY, QUANTITY_MISMATCH, PRICE_MISMATCH, STATUS_MISMATCH, UNKNOWN.

## Unknown Broker Orders

If the broker reports an order that PostgreSQL does not know about:
- DO NOT automatically create a fake TradeIntent.
- DO NOT attribute it to a strategy without evidence.
- Create an UNKNOWN_BROKER_ORDER reconciliation record.
- Flag it for investigation.
- If enough broker metadata safely identifies the order, store the association.
- Never fabricate missing information.

## Unknown Local Orders

If PostgreSQL has an order/execution record but the broker does not show it:
- Do NOT immediately assume the order never existed.
- Consider: order-history availability, broker API timing, data retention, temporary API failure, previous cancellation, previous rejection, network failure.
- Mark it appropriately.
- Do not blindly delete the local record.

## SUBMISSION_UNKNOWN Recovery

Phase 9 can produce SUBMISSION_UNKNOWN. Phase 10 MUST attempt to resolve this state.

Search broker order history/status using available identifiers.

Possible outcomes: confirmed submitted, confirmed rejected, confirmed cancelled, confirmed filled, confirmed partially filled, not found, still unknown.

If confirmed: update PostgreSQL. If still unknown: keep the state unresolved. Do NOT place a replacement order automatically.

## Partial Fills

Reconciliation must correctly handle: requested quantity, filled quantity, pending quantity, average fill price, final status.

Example: Requested = 150, Filled = 75, Pending = 75. The application must not incorrectly mark the position as fully filled.

## Filled Order

If broker reports FILLED but PostgreSQL says SUBMITTED, update the durable state appropriately. Store: filled quantity, average fill price, broker status, timestamps. Do not invent a fill price.

## Cancelled Order

If broker reports CANCELLED, update application order state. Do not treat the order as filled. Do not create a position from a cancelled order.

## Rejected Order

If broker reports REJECTED, store: broker rejection reason, status, timestamp, broker order ID, execution metadata. Do not retry automatically unless a separate future policy explicitly permits it. Do not generate a replacement trade.

## Position Mismatch

Example: PostgreSQL quantity = 75, Broker quantity = 150. This is a serious mismatch.

Do NOT silently overwrite without recording the difference. Create an auditable reconciliation event. Then apply the defined reconciliation policy.

## Local-Only Position

If PostgreSQL says a position is open but broker says no position exists:
Investigate whether: position was closed, broker data is delayed, application state is stale, manual broker action occurred, reconciliation is incomplete.

Do not simply fabricate a broker position.

## Broker-Only Position

If broker shows a position that PostgreSQL does not know about: Treat it as an unexpected broker position.
- Do NOT automatically generate a strategy signal.
- Do NOT automatically close it.
- Do NOT automatically create a new trade.
- Create an auditable discrepancy.

## Manual Broker Actions

The broker account may potentially be modified outside the application (manual order, manual cancellation, manual position closure, manual quantity change).

Phase 10 must detect such differences. Do not assume every broker action originated from the application.

## Reconciliation Policy

Create a centralized reconciliation policy. The policy should determine which discrepancies can be automatically synchronized and which require manual review.

Safe examples for automatic synchronization: broker order status, broker filled quantity, broker average fill price, known broker order timestamps, known position quantity, known broker position state.

Unsafe examples that should require review unless explicitly defined: unknown broker position, unknown broker order, unexpected quantity increase, unexpected manual trade, ambiguous order identity.

Do not invent aggressive automatic corrections.

## No Blind Overwrite

Never blindly execute broker state → overwrite PostgreSQL, or PostgreSQL state → overwrite broker state.

First: detect, classify, record, apply defined policy. This is a reconciliation system, not a blind synchronization script.

## Reconciliation Lock

Use the Phase 3 Redis distributed lock. Prevent two reconciliation processes from modifying the same state simultaneously.

Possible lock scopes: global reconciliation lock, order-specific lock, position-specific lock. Prefer the narrowest appropriate lock.

## Idempotent Reconciliation

Running reconciliation repeatedly must be safe.

Example: Run #1: mismatch detected, state updated. Run #2: same broker state, same local state → Result: MATCHED. No duplicate records should be created unnecessarily.

## Application Restart Recovery

On application startup, reconciliation should be able to detect: unfinished executions, SUBMISSION_UNKNOWN orders, pending orders, partial fills, open positions, stale Redis state, database/broker mismatches.

Do not assume Redis survived correctly. Rebuild runtime state from durable/broker information where necessary.

## Startup Reconciliation

Provide a controlled startup reconciliation mechanism.

Conceptually: Application starts → Database available → Kite authentication available → Broker reconciliation → State consistency verified → Application becomes execution-ready.

Do not automatically enable trading merely because reconciliation succeeded.

## Periodic Reconciliation

If the existing application architecture supports scheduled jobs: provide a controlled periodic reconciliation mechanism. Do not create an uncontrolled infinite loop. Use configurable intervals. Do not poll the broker excessively.

## Manual Reconciliation

Provide a service/API/CLI mechanism where appropriate to trigger reconciliation manually (reconcile orders, reconcile positions, reconcile all). Use the project's existing application interface conventions. Do not create unnecessary frameworks.

## Broker API Usage

Reuse the existing Kite broker adapter. Do not create another Kite client. Use the existing authentication infrastructure. Respect Kite API rate limits. Avoid excessive API calls.

## Error Handling

Handle: Kite unavailable, authentication failure, timeout, rate limiting, network failure, invalid broker response, PostgreSQL unavailable, Redis unavailable, missing order, missing position, unexpected data, serialization failure.

If reconciliation cannot be completed: do NOT claim state is consistent.

## Partial Reconciliation

If orders reconcile successfully but positions fail: Result: PARTIALLY_RECONCILED. Do not report CONSISTENT.

## Redis Failure

If Redis is unavailable: Determine whether reconciliation can safely continue using PostgreSQL and broker state. If Redis is only runtime cache: reconciliation may continue if safe. If Redis contains a critical lock/state required for safety: fail safely. Do not invent unsafe fail-open behavior.

## PostgreSQL Failure

If PostgreSQL is unavailable: Do not perform state-changing reconciliation that cannot be durably recorded. Do not claim successful reconciliation. Return RECONCILIATION_FAILED.

## Audit Trail

Every reconciliation run must be auditable. Record: reconciliation_id, timestamp, environment, scope, result, broker state summary, local state summary, mismatches, resolution, unresolved issues, error information, duration. Do not store credentials.

## Reconciliation History

Create durable reconciliation records where appropriate. Keep historical evidence of: mismatch detected, decision, resolution, unresolved state. Do not overwrite historical reconciliation events.

## State Versioning

Where useful, use updated_at, version, broker timestamp, reconciliation timestamp to avoid applying stale information over newer information.

## Timestamp Handling

Use timezone-aware timestamps. Do not compare naive timestamps with timezone-aware timestamps. Clearly distinguish: broker timestamp, order timestamp, fill timestamp, application timestamp, reconciliation timestamp.

## P&L Reconciliation

Where broker provides P&L: compare against application calculations. Do not blindly replace historical application P&L. Identify realized P&L, unrealized P&L, broker-reported P&L, application-calculated P&L. Document differences.

## Position Average Price

Use broker execution information where broker state is authoritative. Do not calculate average price from incomplete fills. Use Decimal/exact numeric handling.

## Order/Fill History

If broker APIs provide trade/fill history: use it where necessary to resolve partial fills, average price, unknown submission, order status ambiguity. Do not download unnecessary historical data.

## Security

Verify: no credentials hardcoded, no credentials logged, no access tokens logged, no production secrets committed, test environment cannot accidentally use production credentials, reconciliation environment is explicit.

## Logging

Use the existing logger. Important events: RECONCILIATION_STARTED, RECONCILIATION_COMPLETED, RECONCILIATION_MISMATCH, RECONCILIATION_RESOLVED, RECONCILIATION_UNRESOLVED, ORDER_RECONCILED, POSITION_RECONCILED, SUBMISSION_UNKNOWN_RESOLVED, RECONCILIATION_ERROR. Do not log unnecessary full broker payloads. Do not log secrets.

## Testing — Order Reconciliation

Create tests for: matching order, broker-only order, local-only order, status mismatch, quantity mismatch, fill mismatch, price mismatch, unknown submission, filled order, partial fill, cancelled order, rejected order.

## Testing — Position Reconciliation

Test: matching position, broker-only position, local-only position, quantity mismatch, average-price mismatch, open/closed mismatch, zero position.

## Testing — Unknown Submission

Test: SUBMISSION_UNKNOWN, broker confirms submitted, broker confirms rejected, broker confirms filled, broker confirms cancelled, broker still unknown. Ensure NO duplicate order is placed.

## Testing — Idempotency

Run reconciliation twice. Expected: first run mismatch resolved; second run MATCHED. No duplicate state changes.

## Testing — Restart Recovery

Simulate: application crash, Redis restart, database restart, order submission uncertainty, partial fill, pending order. Then run reconciliation. Verify correct recovery behavior.

## Testing — Concurrency

Run two reconciliation attempts concurrently. Verify: locks work, no conflicting updates occur, no duplicate reconciliation actions occur.

## Testing — Failure Modes

Test: Kite unavailable, Kite timeout, authentication failure, rate limit, PostgreSQL unavailable, Redis unavailable, malformed broker response, missing order, missing position. Ensure system does not falsely report consistency.

## Integration Test

Create an isolated integration test using PostgreSQL test database, Redis test instance, mock/fake Kite broker. Test: broker state, local state, reconciliation, database updates, Redis updates, audit records. No production broker access.

## Regression Tests

All previous phases must continue to pass: Phase 1–9.

## README Update

Update README with: Phase 10 architecture, broker authority, PostgreSQL responsibility, Redis responsibility, order reconciliation, position reconciliation, unknown-order handling, unknown-submission handling, partial-fill reconciliation, startup reconciliation, periodic reconciliation, manual reconciliation, mismatch handling, audit trail, Phase 11 boundary.

## Critical Safety Rules

Phase 10 MUST NOT:
- generate trading signals
- change strategy logic
- calculate new trade opportunities
- automatically create replacement trades
- automatically close unexpected positions
- blindly overwrite state
- place new orders as a result of reconciliation
- call order-placement APIs

Reconciliation must be observational and state-correcting only.

**IMPORTANT:** If a mismatch requires a new broker order to correct it: DO NOT place that order. Record REQUIRES_MANUAL_ACTION or equivalent.

## Order Placement Safety Check

Search the complete repository for: place_order, place_order_variety, modify_order, cancel_order, broker order-placement methods. Confirm Phase 10 does NOT call order-placement methods. Phase 9 remains the only component responsible for initiating broker orders.

## Definition of Done

Phase 10 is complete only when:
- Broker order reconciliation implemented.
- Broker position reconciliation implemented.
- Order mismatch detection implemented.
- Position mismatch detection implemented.
- SUBMISSION_UNKNOWN recovery implemented.
- Partial-fill reconciliation implemented.
- Broker-only order detection implemented.
- Local-only order detection implemented.
- Broker-only position detection implemented.
- Local-only position detection implemented.
- Quantity mismatch detection implemented.
- Average-price mismatch detection implemented.
- Status mismatch detection implemented.
- Reconciliation result model implemented.
- Reconciliation history implemented.
- Redis locking implemented.
- Idempotent reconciliation implemented.
- Startup recovery implemented where appropriate.
- Manual reconciliation implemented where appropriate.
- Periodic reconciliation implemented where appropriate.
- Error handling implemented.
- Audit trail implemented.
- Unit tests implemented.
- Failure-mode tests implemented.
- Concurrency tests implemented.
- Integration tests implemented.
- All Phase 1–9 tests pass.
- README updated.
- NO new order generation.
- NO strategy changes.
- NO backtesting.
- NO optimization.
- NO Phase 11+ functionality.

## Final Report

At the end, report: Phase 1–9 architecture reviewed, Phase 10 architecture, broker authority model, order reconciliation design, position reconciliation design, SUBMISSION_UNKNOWN recovery, partial-fill handling, unknown broker order handling, unknown broker position handling, mismatch categories, automatic reconciliation rules, manual-review rules, PostgreSQL changes, Redis changes, startup recovery, periodic reconciliation, manual reconciliation, distributed locking, idempotency, audit trail, error handling, files created, files modified, tests created, test results, Phase 1–9 regression results, security checks, known limitations, unresolved architectural decisions, TODOs for Phase 11.

**IMPORTANT:** STOP AFTER PHASE 10. Do NOT implement Phase 11. Do NOT implement backtesting. Do NOT implement paper trading. Do NOT implement optimization. Do NOT implement production deployment. Do NOT generate trading signals. Do NOT place new broker orders. Do NOT automatically close unexpected broker positions. Do NOT change strategy logic.

**Final Phase 10 Output:**

```
BROKER
  ↓
RECONCILIATION ENGINE
  ↓
COMPARE
  ├── PostgreSQL
  └── Redis
  ↓
MATCH / MISMATCH
  ↓
RECONCILE SAFE STATE
  ↓
AUDIT RESULT
  ↓
CONSISTENT STATE
```

PHASE 11 starts from a reconciled trading system.
