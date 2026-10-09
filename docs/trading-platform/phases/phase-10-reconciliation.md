# Phase 10 — Broker Reconciliation & State Consistency

Phase 10 detects and safely resolves differences between the Kite
**broker** (authoritative for actual executions/fills/positions),
**PostgreSQL** (the durable application record), and **Redis** (fast
runtime cache). It is purely observational/state-correcting: it never
generates a signal, never changes strategy/risk logic, never sizes a
position, and never calls `Broker.place_order()`/`modify_order()`/
`cancel_order()` -- Phase 9 (`OrderExecutionEngine`) remains the only
component that ever submits a broker order. Confirmed by grepping the
whole repository for every order-placement call site.

```text
Kite Broker (Broker.get_orders() / get_positions())
  -> ReconciliationEngine (Phase 3 DistributedLock: "reconciliation:global")
  -> OrderReconciler              PositionReconciler
       compares against             compares against
       PostgreSQL `orders`          PostgreSQL `positions`
       (+ tag/broker_order_id       (existing row, else this
        matching for               application's own recorded
        SUBMISSION_UNKNOWN)        fills -- never broker data)
  -> ReconciliationPolicy (safe auto-sync vs. REQUIRES_MANUAL_ACTION)
  -> PostgreSQL update (orders/positions) + audit trail
     (`reconciliation_runs` / `order_reconciliation_records` /
      `position_reconciliation_records`)
  -> best-effort Redis runtime-cache refresh (ExecutionRuntimeStateStore /
     RiskPositionStateStore) -- Redis is never authoritative
```

**Broker authority:** for actual executions/fills/positions, the broker
is authoritative. A local/broker difference is never assumed away in
either direction (`quantity=0` locally vs. `quantity=75` at the broker is
a detected discrepancy, not evidence PostgreSQL is "probably right").

**Order reconciliation (`reconciliation/order_reconciler.py`):** matches
a persisted `Order` row to a broker order by `broker_order_id` first,
falling back to the exact same deterministic tag Phase 9 embeds in every
broker request (`execution/order_tag.py`, `{strategy-prefix}-{execution_id
hex[:12]}`) when `broker_order_id` is unknown -- this is how
`SUBMISSION_UNKNOWN` is resolved without ever fabricating an association.
Outcomes: `MATCHED`, `BROKER_NEW` (broker order no local record/tag
explains -- flagged, never attributed to a strategy), `LOCAL_NEW` (a
local order the broker's current order book no longer shows -- flagged,
never deleted), `STATUS_MISMATCH`, `QUANTITY_MISMATCH` (always manual --
the requested quantity must never differ), `FILL_MISMATCH` (including a
broker fill **regression**, always manual), `PRICE_MISMATCH` (including a
filled order with no valid broker average price, always manual),
`SUBMISSION_UNKNOWN_RESOLVED`/`SUBMISSION_UNKNOWN_STILL_UNRESOLVED`.
Safe-to-auto-sync fields (broker order status, filled quantity, average
fill price) are controlled by `ReconciliationPolicy`
(`RECONCILIATION_AUTO_SYNC_ORDER_*`, all default `true`); an illegal/
backward status transition is never applied even when policy allows
auto-sync. Partial fills are tracked exactly (`filled`/`pending`
quantity); a `150`-requested/`75`-filled order is never marked fully
filled.

**Position reconciliation (`reconciliation/position_reconciler.py`):**
compares the broker's net position per instrument against the existing
`positions` row (if any), or -- only when no row exists yet -- an
aggregate computed purely from this application's own already-recorded
Phase 9 order fills (never from broker data, never invented). Outcomes:
`MATCHED`, `BROKER_ONLY` (an unexpected broker position -- always
`REQUIRES_MANUAL_ACTION`; never attributed to a strategy, never closed,
never a new trade), `LOCAL_ONLY` (PostgreSQL shows a position the broker
does not -- always manual, never auto-closed/fabricated), `QUANTITY_MISMATCH`,
`PRICE_MISMATCH`, `STATUS_MISMATCH`. An **unsafe quantity change** --
broker quantity larger in magnitude than local, or a sign flip -- is
always `REQUIRES_MANUAL_ACTION` and is never silently overwritten
(`reconciliation/policy.py::is_safe_position_quantity_sync`); only a
broker quantity the *same or smaller* in the *same* direction (a close/
partial close) is eligible for auto-sync, alongside average price (the
broker is authoritative for actual fill price) and open/closed status.

**Reconciliation policy (`reconciliation/policy.py`):** a single
centralized, explicit allow-list of what may auto-sync. Every flag
defaults `true` and only ever governs a category the specification calls
"safe" (known broker order status/fill/price, known broker position
quantity/price/state); unknown-broker-order, unknown-broker-position, and
unsafe-quantity-increase categories are **not** governed by any flag --
they always require manual action. Nothing here invents an aggressive
automatic correction.

**Distributed locking & idempotency:** `ReconciliationEngine` acquires
the existing Phase 3 `DistributedLock` (`reconciliation:global`, TTL
`RECONCILIATION_LOCK_TTL_SECONDS=120`) before touching any state; a
concurrent reconciliation attempt fails closed as
`RECONCILIATION_FAILED` rather than racing. Running reconciliation
repeatedly is safe: a second run against unchanged broker/local state
reports `MATCHED`/`CONSISTENT` and writes no additional records --
`order_reconciliation_records`/`position_reconciliation_records` are only
ever written for a non-`MATCHED` outcome.

**Result model & audit trail:** `ReconciliationResult`
(`reconciliation_id`, `timestamp`, `environment`, `scope`, `status`,
checked/detected/resolved/unresolved counts, `errors`, `duration_seconds`)
is always returned, and a matching append-only `reconciliation_runs` row
(plus per-mismatch `order_reconciliation_records`/
`position_reconciliation_records` rows) is persisted every run --
including failed ones -- via new migration `0009_reconciliation`.
Historical reconciliation rows are never overwritten or deleted.
`status` is one of `CONSISTENT`, `RECONCILED`, `MISMATCH_DETECTED`,
`PARTIALLY_RECONCILED`, `RECONCILIATION_FAILED` -- a scope that only
partially succeeds (e.g. orders reconciled, positions failed) is reported
`PARTIALLY_RECONCILED`, never `CONSISTENT`; if the durable audit write
itself fails, the result is downgraded to `RECONCILIATION_FAILED`
regardless of what was computed -- an un-auditable run is never reported
as successful.

**Startup / periodic / manual reconciliation:** `TradingEngine` builds a
`ReconciliationEngine` during `initialize()` (same pattern as Phase 9's
`OrderExecutionEngine` -- built eagerly, never auto-invoked).
`run_startup_reconciliation()` and `run_manual_reconciliation(scope)` are
explicit calls (see `scripts/reconcile.py` for a CLI entry point: `python
scripts/reconcile.py --scope all|orders|positions`); reconciliation
succeeding never automatically enables trading. `start_periodic_reconciliation()`
starts an opt-in `PeriodicReconciliationScheduler` background thread only
if `RECONCILIATION_INTERVAL_SECONDS` is explicitly set (`None` by
default); it is a plain bounded sleep loop, not an uncontrolled poll, and
`shutdown()` stops it.

**Error handling:** a broker failure fetching orders/positions, or a
PostgreSQL failure persisting the audit trail, is caught per-scope and
never silently reported as success; `ALL`-scope reconciliation reports
`PARTIALLY_RECONCILED` if only one of orders/positions failed, or
`RECONCILIATION_FAILED` if both (or the requested single scope) failed.
Redis failures acquiring the lock also fail closed as
`RECONCILIATION_FAILED`; Redis failures refreshing the best-effort
runtime cache afterward are logged and never fail the run (Redis is
never authoritative).

**Configuration:** `RECONCILIATION_LOOKBACK_HOURS=48` (bounds the local
order pool used for broker tag/id matching, mirroring Kite's own
`orders()` endpoint which only returns the current trading day),
`RECONCILIATION_LOCK_TTL_SECONDS=120`,
`RECONCILIATION_INTERVAL_SECONDS` (unset/`None` by default),
`RECONCILIATION_AUTO_SYNC_ORDER_STATUS`/`_ORDER_FILL_QUANTITY`/
`_ORDER_AVERAGE_FILL_PRICE`/`_POSITION_QUANTITY`/`_POSITION_PRICE`/
`_POSITION_STATUS` (all default `true`). All also read from the existing
single Secrets Manager secret via the generic per-field loop.

**Persistence/migration:** `0009_reconciliation` adds `reconciliation_runs`,
`order_reconciliation_records`, `position_reconciliation_records`. Apply
`alembic upgrade head` before using Phase 10.

**Tests and boundary:** `tests/unit/test_order_reconciler.py`,
`test_position_reconciler.py`, and `test_reconciliation_engine.py` use
SQLite/fakeredis/fake brokers only, including a lock-contention/
concurrency test and a PostgreSQL-unavailable failure test, and an
explicit assertion that `Broker.place_order`/`modify_order`/`cancel_order`
are never called during reconciliation. The gated
`tests/integration/test_reconciliation_integration.py` uses a fake broker
against real PostgreSQL/Redis and cleans up every row/key it creates.
Phase 10 does not generate trading signals, size positions, or place any
order to correct a detected mismatch -- an unresolved mismatch is always
left for manual action; that boundary is enforced by the policy module,
not by convention alone.

See the original prompt: [specs/10-phase10.md](../specs/10-phase10.md).
