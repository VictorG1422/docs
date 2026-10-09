# Phase 9 — Order Execution

Phase 9 accepts an already persisted `RISK_APPROVED` Phase 8 `TradeIntent`.
It never creates signals, changes strategy/risk decisions, changes the
approved quantity, or automatically runs on a signal. `TradingEngine`
exposes an explicit `execute_trade_intent(intent)` handoff after
initialization; the main `Application`/strategy pipeline does not yet
automatically create and execute intents.

```text
Phase 8 TradeIntent (must be persisted and risk-approved)
  -> validate intent + persisted signal/risk decision/instrument
  -> execution permission + environment gate
  -> Redis per-intent lock + dedup, PostgreSQL unique intent record
  -> durable SUBMITTING state
  -> BrokerOrderAdapter -> KiteBrokerAdapter -> existing Kite Broker API
  -> broker order ID -> durable order state / Redis runtime cache
  -> controlled explicit status refresh -> OPEN / PARTIAL / FILLED / terminal state
```

**Execution safety:** `ORDER_EXECUTION_ENABLED` defaults to `false`.
Live broker submission additionally requires `APP_ENV=production` and
`TRADING_MODE=live`; Kite credentials or live mode alone never enable
orders. Paper/test execution requires a non-production environment and
the fake adapter; automated tests never use production credentials or
real broker order APIs. The legacy signal-based `OrderManager` continues
to use its paper handler; `LiveExecutionHandler` now fails closed and
cannot bypass the Phase 9 TradeIntent gate.

**Validation:** Phase 9 requires an approved persisted intent, a matching
signal and approved risk decision, instrument metadata, matching symbol
and token, BUY/SELL direction, a positive integer quantity exactly equal
to the approved quantity and aligned to the stored lot size, supported
order/product/validity values, and valid Decimal prices aligned to the
instrument tick size. It rejects mismatches rather than adjusting them.
SL/SL-M entry orders require an explicit entry trigger. Phase 8 stop-loss
and target values are preserved in the durable order metadata; Phase 9
does not create separate protective orders.

**Duplicate/failure handling:** Redis uses the existing namespaced
`DistributedLock` and `DeduplicationStore`; PostgreSQL's unique
`trade_intent_id` on the existing `orders` table remains authoritative.
The order is committed as `SUBMITTING` before the external broker call.
A known rejection is terminal. A timeout, network error, process restart
while submitting, or inability to durably record a broker acknowledgement
becomes `SUBMISSION_UNKNOWN`; Phase 9 never resubmits it automatically.
This is intentionally at-most-once, not a claim of exactly-once delivery.

**Lifecycle/state:** PostgreSQL stores append-only `execution_attempts`
for every call (including disabled-gate/validation/lock rejections), plus
the durable order after validation. PostgreSQL orders store execution ID,
intent/signal IDs,
broker order ID, approved quantity, fill quantity, average fill price,
execution state, errors, retry count, and strategy/risk versions. Redis
contains only expiring recent-state cache and locks; it is never the
source of truth. Broker acknowledgement produces `SUBMITTED`, not
`FILLED`. `refresh_status(intent_id)` refreshes only a known broker order
and tracks OPEN, PARTIALLY_FILLED, FILLED, CANCELLED, or REJECTED; partial
fills update the Redis position lifecycle as PARTIAL, and only a
confirmed full fill promotes it to OPEN. This is not broker reconciliation.

**Configuration:** `ORDER_EXECUTION_ENABLED=false` by default;
`EXECUTION_ORDER_TYPE=MARKET`, `EXECUTION_PRODUCT=MIS`,
`EXECUTION_VALIDITY=DAY`, optional `EXECUTION_MAX_QUANTITY`, and
`EXECUTION_LOCK_TTL_SECONDS=30`. These variables are also read from the
existing single Secrets Manager secret. Set live permission only after
reviewing Phase 12 readiness; it requires all of `APP_ENV=production`,
`TRADING_MODE=live`, `ORDER_EXECUTION_ENABLED=true`, and valid Kite auth.

**Persistence/migration:** `0007_phase9_execution` adds strategy version
to persisted TradeIntents; `0008_order_execution` extends the existing
`orders` table with intent/execution linkage and lifecycle fields. Apply
`alembic upgrade head` before using Phase 9.

**Tests and boundary:** `tests/unit/test_order_execution.py`,
`test_broker_order_adapter.py`, and `test_live_executor_safety.py` use
SQLite/fakeredis/fake brokers only. The gated
`tests/integration/test_phase9_execution_integration.py` also uses a fake
adapter and never submits a real order. Phase 9 does not reconcile
unknown submissions, broker order history, broker positions, or
PostgreSQL/Redis consistency; that begins in Phase 10.

See the original prompt: [specs/09-phase9.md](../specs/09-phase9.md).
