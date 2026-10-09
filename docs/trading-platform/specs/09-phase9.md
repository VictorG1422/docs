# PHASE 9 - ORDER EXECUTION ENGINE

## Project Roadmap

Completed:

- PHASE 1 - Core Application Foundation
- PHASE 2 - PostgreSQL Database
- PHASE 3 - Redis State & Cache Layer
- PHASE 4 - Kite Connect & Market Data Foundation
- PHASE 5 - Instrument & Option-Chain Foundation
- PHASE 6 - Market Data & Technical Analysis
- PHASE 7 - Trading Strategy & Signal Engine
- PHASE 7A - Actual Strategy Specification
- PHASE 8 - Risk & Position Management

This phase: **PHASE 9 - ORDER EXECUTION ENGINE**

Upcoming: PHASE 10 - Broker Reconciliation; PHASE 11 - Backtesting & Paper Trading; PHASE 12 - Production & Live-Trading Readiness; PHASE 13 - Optimization.

## Scope and Objective

Implement Phase 9 only. Phase 8 produces a risk-approved `TradeIntent`; Phase 9 validates and converts it into a broker order through the existing Kite integration, then records order/execution lifecycle state. Do not implement Phase 10 or later.

Architecture:

```text
Strategy -> Signal -> Risk Engine -> RISK-APPROVED TradeIntent
  -> Phase 9 Order Execution Engine -> Broker Adapter -> Kite Order API
  -> Broker Order ID -> Order Lifecycle Tracking -> PostgreSQL + Redis runtime state
```

Phase 9 MUST NOT generate strategy signals, modify strategy rules, calculate new indicators, invent position sizes or risk limits, perform complete broker reconciliation, implement backtesting, or implement optimization.

## Inspect Before Implementation

Inspect the complete project and review configuration/logging, PostgreSQL, Redis, Kite integration/authentication, instruments/options, market data/indicators, strategy/signals, strategy specification, risk engine/TradeIntent, position state, distributed locks, deduplication, tests, and existing order functionality. Search the whole repo for `place_order`, `modify_order`, `cancel_order`, order status, execution, and position updates. Do not duplicate existing functionality.

Before implementation, explain briefly: current architecture; TradeIntent handoff; Kite communication; lifecycle; PostgreSQL and Redis responsibilities; duplicate protection; failure/retry handling; Phase 10 boundary; files to create/modify.

## Validation and Safety Gate

Before placing any order, validate:

- Intent exists and has `RISK_APPROVED` status, with valid intent ID and signal ID.
- Referenced signal and instrument exist; instrument token, tradingsymbol, and exchange are valid.
- Direction is valid; quantity is a positive integer, respects the instrument lot size, and exactly matches the approved intent. Never silently alter it.
- Order type/product are supported; required price and trigger price are valid and use instrument tick size/Decimal precision.
- Strategy name/version and risk-policy version exist.
- Intent has not already been executed.

Any critical validation failure means no broker call and a clear rejection reason.

Final gate:

```text
TradeIntent -> validation -> execution enabled? -> duplicate check
 -> execution lock -> final validation -> Kite Order API
```

No gate failure may call Kite.

## Environment and Live-Execution Safety

Reuse environment configuration. Separate development, paper/test, and production. Kite authentication alone MUST NOT enable live orders. Add an explicit order-execution flag, default disabled. Tests must use fake/mock adapters and must not use production credentials.

Before a real Kite order, require production environment, valid Kite authentication, explicit execution enablement, a risk-approved intent, valid parameters, duplicate check passed, execution lock acquired, and all final safety checks passed. If any condition fails, do not place the order.

## Broker Adapter and Order Request

Reuse the existing Kite abstraction; extend it cleanly if needed. Architecture:

```text
OrderExecutionEngine -> BrokerOrderAdapter -> KiteBrokerAdapter -> Kite API
```

Keep raw Kite SDK details out of the execution engine so another broker can be added later. Only execution-layer/broker-adapter code may call broker order-placement methods; strategy and risk code must never call them directly.

Create a structured internal `OrderRequest` with only fields the project actually needs, sourced from `TradeIntent`: intent/signal IDs, instrument token, tradingsymbol, exchange, transaction type, quantity, order type, product, validity, price/trigger price, tag, strategy name/version, and risk-policy version.

Map approved intent direction without inference or modification: BUY -> BUY and SELL -> SELL. Reject unapproved intents. Support only required order types (potentially MARKET, LIMIT, SL, SL-M); do not add unnecessary features. Source product from existing configuration/intent; do not scatter hardcoded product defaults. Validate supported products.

Validate positive integer quantity, lot size, instrument, and configured execution limits; never resize. For limit/stop orders, validate price, trigger price, tick size and price relationships using instrument metadata and Decimal arithmetic; never invent universal tick sizes or silently lose precision. Use a safe broker tag identifying strategy/intent/execution, respect broker length limits, and never include secrets.

## Idempotency, Locking, and Failure Handling

The same TradeIntent must never accidentally create multiple broker submissions. Protect duplicate events, retries, process restarts, concurrency, and network timeouts using PostgreSQL execution records, Redis deduplication, and the existing TTL-bounded Redis distributed lock. Lock flow: acquire -> inspect durable execution state -> submit -> persist broker order ID -> update runtime state -> release. Avoid global locks and permanent locks.

PostgreSQL is durable source of truth; Redis is runtime acceleration only. PostgreSQL transactions cannot be atomic with Kite API requests, so handle the external side-effect boundary carefully.

Critical ambiguity: Kite may accept an order while its response is lost. Do not blindly retry. Persist `SUBMISSION_UNKNOWN` (or equivalent), do not automatically resubmit, and leave reconciliation to Phase 10. At-most-one submission per intent unless later reconciliation proves the first definitely failed. Retry only when it is clear the broker did not receive the order.

Handle authentication failure/expired token, invalid instrument/quantity/price, insufficient margin, market closed, broker rejection, rate limits, timeout/network failure, temporary broker failure, and unknown errors with clear internal categories. Never expose credentials/tokens/passwords. Avoid aggressive retries/polling; use controlled backoff only when safe. Do not retry unknown submissions.

## Order Lifecycle

Use clear execution/order states, as supported by the architecture: CREATED, VALIDATING, SUBMITTING, SUBMITTED, OPEN, PARTIALLY_FILLED, FILLED, CANCEL_PENDING, CANCELLED, REJECTED, FAILED, SUBMISSION_UNKNOWN. Enforce legal transitions. A Kite acceptance response is not a fill.

When Kite returns a broker order ID, persist it immediately and associate it with intent, signal, execution, instrument, quantity, timestamp, strategy version, and risk-policy version. Where required, retrieve/update broker status with controlled polling and stop at terminal states. Do not implement complete reconciliation.

Track ordered, filled, pending quantities and average fill price accurately. Do not assume order quantity equals filled quantity. Do not mark a position fully OPEN merely because an order was submitted: submission is pending; partial fill yields partial position; full fill yields open position; rejected/cancelled unfilled order yields no filled position. Preserve TradeIntent stop-loss/target values exactly; do not calculate or submit separate protective orders unless explicitly required by an existing specification.

## Persistence and Runtime State

PostgreSQL stores durable order/execution records, with appropriate models/repositories/migrations and transaction boundaries. Consider execution ID, intent ID, signal ID, broker order ID, instrument, exchange, transaction, ordered/filled/pending quantities, order type/product/prices, status/messages, timestamps, fill price, strategy name/version, and risk-policy version. Avoid duplicate models/repositories where an existing table can be extended cleanly.

Redis stores execution locks, deduplication, recent runtime status, and temporary broker status cache. It is never authoritative.

## Audit and Logging

Use the project logger. Add events as appropriate: `ORDER_EXECUTION_STARTED`, `ORDER_VALIDATION_FAILED`, `ORDER_SUBMISSION_STARTED`, `ORDER_SUBMITTED`, `ORDER_SUBMISSION_UNKNOWN`, `ORDER_STATUS_UPDATED`, `ORDER_PARTIALLY_FILLED`, `ORDER_FILLED`, `ORDER_REJECTED`, `ORDER_CANCELLED`, and `ORDER_EXECUTION_ERROR`.

Audit intent ID, signal ID, execution ID, broker order ID, timestamp, state, error category, retry count, environment, strategy version, and risk-policy version. Never log credentials, access tokens, API secrets, passwords, or sensitive configuration.

## Development and Test Mode

Provide a safe fake broker adapter for `TradeIntent -> ExecutionEngine -> Fake Broker` tests. No developer should need to comment out code to prevent live orders. Any real-Kite integration test must be disabled by default and require explicit manual activation.

## Required Tests

Validation tests: valid/invalid intent, rejected risk status, missing instrument, invalid quantity/non-integer/lot size, invalid price/order type, execution disabled, invalid environment, missing authentication, duplicate intent.

Idempotency tests: first execution invokes broker once; repeated intent does not; concurrent execution invokes at most once; restart/retry after known submission does not duplicate. Test network failure before submission versus accepted-by-Kite/response-lost; the latter must become `SUBMISSION_UNKNOWN` and must not retry.

Lifecycle tests: CREATED, SUBMITTING, SUBMITTED, OPEN, PARTIALLY_FILLED, FILLED, CANCELLED, REJECTED, FAILED, SUBMISSION_UNKNOWN and legal transitions. Partial-fill tests cover zero/partial/full quantity, average fill price, remaining quantity, and position-state behavior.

Adapter tests: correct request mapping; invalid request rejected before broker call; broker success/error mappings; timeout and unknown submission.

Add an isolated PostgreSQL/Redis integration test using only a fake/mock broker, gated by the project’s integration-test convention. It MUST NOT place real orders. Preserve all Phase 1-8 tests.

## Documentation and Security Review

Update README with Phase 9 architecture, intent-to-execution flow, adapters, lifecycle/states, duplicate protection, locks, idempotency, unknown submissions, partial fills, PostgreSQL/Redis roles, fake execution, explicit live gate, and Phase 10 boundary.

Before completion, verify no credentials are hardcoded/logged, tests use no production credentials, execution is disabled by default, environment separation exists, and automated tests cannot submit production orders. Search the complete repository for `place_order`, `modify_order`, `cancel_order`, Kite order calls, and broker execution calls. Confirm strategy/risk code cannot place orders and identify exactly where order placement is permitted.

## Phase 10 Boundary

Stop after Phase 9. Do not implement complete broker order-history reconciliation, broker position reconciliation, missing/unknown-order reconciliation, position mismatches, restart reconciliation, or broker-vs-PostgreSQL/Redis consistency. Those belong to Phase 10. Also do not implement backtesting, production deployment, optimization, strategy changes, or new risk rules.

## Definition of Done

Phase 9 is complete only when intent validation, safety gate, environment separation, explicit disabled-by-default execution flag, OrderRequest, broker adapter, Kite adapter, execution engine, safe broker submission, duplicate protection, Redis lock, broker ID persistence, lifecycle and partial-fill state, PostgreSQL durable records, Redis runtime state, unknown-submission handling, conservative retries, broker error mapping, authentication handling, Decimal price/tick-size checks, quantity/lot-size checks, fake execution, unit/concurrency/failure/integration tests, and README update are complete; all Phase 1-8 regressions pass; no reconciliation or Phase 10+ work is introduced.

Final flow:

```text
PHASE 8: RISK-APPROVED TRADE INTENT
  -> PHASE 9: ORDER EXECUTION ENGINE
  -> KITE BROKER
  -> BROKER ORDER ID
  -> ORDER STATUS / FILL
  -> POSTGRESQL + REDIS
  -> STOP

PHASE 10 starts with broker reconciliation.
```

## Final Report Checklist

Report Phase 1-8 architecture review; files created/modified; TradeIntent validation; execution safety gate; broker/Kite adapters; OrderRequest; lifecycle; duplicate protection/locking/idempotency; unknown submission and retry handling; partial fills; PostgreSQL/Redis changes; position-state behavior; broker error mapping; mock execution; live execution config; tests/results; Phase 1-8 regressions; security checks; exact order-placement capability location; confirmation strategy/risk cannot place directly; limitations; Phase 10 TODO boundary.
