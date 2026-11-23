# Phase 3 — Redis State & Cache Layer

## Query 1 — Local environment info

My postgres DB is running local and redis also runninhg
postgres:
host: localhost
port: 542
user : postgres
password : postgres
redis:
host: localhost
port: 6379

## Query 2 — Trigger

Run and create databases and required items for phase 3

## Query 3 — Full Phase 3 prompt

Phase 1 and Phase 2 have been completed.

PostgreSQL has been created and configured.

Redis has also been created and is available.

Now implement **PHASE 3 ONLY: Redis infrastructure, state management, caching, and distributed-safety primitives**.

Do NOT implement Kite Connect, WebSocket, instrument downloading, option-chain logic, trading strategy, order execution, SL/Target, or live trading in this phase.

---

# **1. Inspect Existing Project First**
Before making changes:

- Inspect the complete repository.
- Review Phase 1 configuration.
- Review Phase 1 logging.
- Review Phase 1 health checks.
- Review Phase 2 PostgreSQL/database implementation.
- Reuse the existing architecture.
- Do not duplicate configuration or logging systems.
- Do not rewrite working code unnecessarily.

First briefly explain:

1. How Redis will integrate with the existing architecture.
2. Which files will be created.
3. Which files will be modified.

Then implement Phase 3.

---

# **2. Redis Technology**
Use a mature Python Redis client.

Prefer:

```
redis-py
```

Use the existing Python architecture and configuration style.

Do not introduce unnecessary Redis frameworks.

---

# **3. Redis Configuration**
Extend the existing configuration system with:

```
REDIS_HOST
REDIS_PORT
REDIS_PASSWORD
REDIS_DB
REDIS_SSL
REDIS_SOCKET_TIMEOUT
REDIS_CONNECT_TIMEOUT
```

Use sensible defaults where appropriate.

Example:

```
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_SSL=false
```

Do not hardcode credentials.

Do not create a second configuration system.

---

# **4. Redis Connection Manager**
Create a clean Redis connection abstraction.

Conceptually:

```
Application
    ↓
Redis Manager
    ↓
Redis Connection Pool
    ↓
Redis
```

Requirements:

- connection pooling
- connection timeout
- socket timeout
- graceful connection close
- error handling
- reconnect support through the client where appropriate

Do not create one new Redis connection for every operation.

Use a connection pool.

---

# **5. Redis Health Check**
Extend the existing health-check framework.

Add:

```
Redis health check
```

The health check should verify that Redis is reachable.

Conceptually:

```
{
  "application": "healthy",
  "postgresql": "healthy",
  "redis": "healthy"
}
```

If Redis is unavailable:

```
Redis: UNHEALTHY
```

Do not expose Redis credentials.

Do not crash the entire application merely because a health check fails.

---

# **6. Redis Key Design**
Create a centralized Redis key strategy.

DO NOT scatter raw Redis key strings throughout the application.

Create a key builder/helper.

Use namespaced keys.

Conceptually:

```
trading:{environment}:market:{instrument_token}
trading:{environment}:position:{instrument_token}
trading:{environment}:strategy:{strategy_name}
trading:{environment}:lock:{name}
trading:{environment}:tick:{instrument_token}
```

The exact naming convention can be improved if needed, but it must be:

- predictable
- documented
- collision-resistant
- environment-aware

For example:

```
development
paper
production
```

must not accidentally share state.

---

# **7. Redis State Manager**
Create a reusable state manager abstraction.

It should support operations such as:

```
set()
get()
delete()
exists()
expire()
set_json()
get_json()
```

If using JSON serialization, handle serialization/deserialization safely.

Do not store arbitrary Python objects using unsafe serialization such as pickle unless there is a compelling reason.

Prefer JSON or another explicit serialization format.

---

# **8. TTL Support**
Redis should not accumulate stale temporary data forever.

Support TTL for temporary state.

For example:

```
latest tick
temporary locks
temporary cache
short-lived market state
```

The TTL should be configurable where appropriate.

Do not blindly apply TTL to data that must survive until trading recovery.

Clearly distinguish:

```
temporary/cache state
```
from:

```
persistent/recovery-critical state
```

---

# **9. Market Data State**
Prepare Redis for future real-time market data.

Create an abstraction that can eventually store:

```
instrument_token
last_price
volume
open
high
low
close
open_interest
timestamp
```

However:

IMPORTANT:

Do NOT implement Kite WebSocket or actual tick ingestion in Phase 3.

Only create the Redis storage interface.

The future flow will be:

```
Kite WebSocket
       ↓
Tick Processor
       ↓
Redis Market State
       ↓
Strategy
```

---

# **10. Latest Tick Storage**
Create an abstraction such as:

```
MarketStateStore
```

It should eventually support:

```
set_latest_tick()
get_latest_tick()
delete_latest_tick()
```

Use instrument token as the primary lookup identifier.

Do not subscribe to any instruments yet.

Do not connect to Kite yet.

---

# **11. Strategy State**
Create a generic strategy-state abstraction.

Eventually the strategy may need to maintain:

```
current signal
last signal
last entry
last exit
trade count
cooldown
strategy-specific state
```

Create an interface such as:

```
StrategyStateStore
```

It should allow strategy-specific state to be stored and retrieved.

Do not implement actual strategy rules.

Do not decide what indicators or signals the strategy will use.

---

# **12. Position State**
Create a Redis position-state abstraction.

It should eventually support fast access to:

```
instrument
quantity
average entry price
current price
position status
stop loss
target
strategy
updated timestamp
```

IMPORTANT:

Redis is NOT the authoritative permanent position database.

The durable position remains in PostgreSQL and/or broker state.

Redis provides fast access to current state.

---

# **13. Order/Execution Locks**
This is important.

Create a Redis distributed-lock mechanism that can eventually prevent duplicate order execution.

Conceptually:

```
Strategy generates signal
        ↓
Acquire execution lock
        ↓
Check current state
        ↓
Create order
        ↓
Release lock
```

The lock must have a TTL so a crashed process cannot permanently lock the system.

Example conceptual API:

```
acquire_lock(name, ttl)
release_lock(name)
```

Do not implement actual order placement.

Only implement the locking primitive.

---

# **14. Duplicate Signal Protection**
Create a mechanism for detecting duplicate processing.

For example, support a unique signal/event identifier.

Conceptually:

```
signal_id
    ↓
Redis
    ↓
Already processed?
    ├── YES → ignore
    └── NO  → mark processed
```

The exact strategy signal format will be implemented later.

Do not invent strategy logic.

---

# **15. Atomic Operations**
Where race conditions are possible, use Redis atomic operations.

For example:

```
SET NX
```
for lock acquisition where appropriate.

Do not implement:

```
GET
then
SET
```
for operations that must be atomic.

Think about concurrent processing even though the initial application is a modular monolith.

---

# **16. Cache Abstraction**
Create a generic cache interface.

Conceptually:

```
cache.set(key, value, ttl)
cache.get(key)
cache.delete(key)
```

The abstraction should allow future components to cache:

- instrument lookups
- option contracts
- market state
- configuration-like runtime data

Do not cache everything automatically.

Caching must have an explicit purpose.

---

# **17. Serialization**
Use explicit serialization.

For structured data, prefer JSON-compatible representations.

Handle:

- datetime
- Decimal
- enums

consistently.

Do not silently lose precision for prices.

Pay particular attention to:

```
price
quantity
P&L
strike
```

Use appropriate numeric representations.

Do not casually convert financial values to floating point when exact representation matters.

---

# **18. Redis Error Handling**
Handle:

- Redis unavailable
- timeout
- connection failure
- invalid data
- serialization failure

Redis failures should be logged clearly.

Do not silently swallow errors.

Where Redis is only being used as a cache, the application should be able to distinguish:

```
cache unavailable
```
from:

```
critical state unavailable
```

Do not make arbitrary fail-open/fail-closed decisions for trading operations yet.

Those decisions will be made when the execution/risk architecture is implemented.

---

# **19. PostgreSQL vs Redis Responsibility**
Document this clearly.

### **PostgreSQL**
Use for durable information:

```
instruments
orders
positions
trades
signals
historical records
```

### **Redis**
Use for fast temporary/runtime state:

```
latest ticks
current market state
active runtime state
temporary locks
deduplication
cache
strategy runtime state
```

The architecture should follow:

```
             ┌──────────────┐
             │ PostgreSQL   │
             │ Durable      │
             └──────┬───────┘
                    │
                    │
             ┌──────▼───────┐
             │ Trading App  │
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │ Redis        │
             │ Fast State   │
             └──────────────┘
```

Redis must NOT become the only source of truth.

---

# **20. Testing**
Create unit tests for the Redis layer.

Tests should cover:

### **Connection**

- valid configuration
- invalid configuration
- unavailable Redis

### **State**

- set/get
- JSON set/get
- delete
- exists
- expiration

### **Market state**

- store latest tick
- retrieve latest tick
- update latest tick

### **Strategy state**

- store state
- retrieve state
- update state

### **Position state**

- store
- retrieve
- update

### **Locks**
Test:

```
first acquisition → success
second acquisition → failure
release → possible acquisition
TTL expiration → lock becomes available
```

### **Deduplication**
Test:

```
new event → accepted
same event → rejected/duplicate
```

Use a mocked Redis or isolated test Redis.

Do NOT connect tests to a production Redis instance.

---

# **21. Integration Testing**
If practical, add an integration test that runs against a dedicated test Redis instance.

Do not require developers to accidentally connect to production Redis.

The test environment must be isolated.

---

# **22. Logging**
Use the existing Phase 1 logger.

Log important Redis events:

```
REDIS_CONNECTED
REDIS_DISCONNECTED
REDIS_HEALTH_CHECK
REDIS_ERROR
REDIS_LOCK_ACQUIRED
REDIS_LOCK_FAILED
```

Do not log:

- passwords
- secrets
- full sensitive configuration
- unnecessary high-frequency tick payloads

IMPORTANT:

Do not log every future market-data tick at INFO level.

That would create an absurd amount of noise.

---

# **23. Performance Considerations**
Redis will eventually sit on the real-time trading path.

Keep operations lightweight.

Avoid:

- unnecessary serialization
- excessive round trips
- scanning the entire keyspace
- blocking operations
- large values
- storing unnecessary historical data

Do not prematurely optimize with Lua scripts or complex Redis structures unless there is a real requirement.

---

# **24. README Update**
Update README with:

## **Redis Setup**
Explain:

- Redis requirement
- required environment variables
- how to start Redis locally
- how to verify connectivity

## **Redis Responsibilities**
Document:

```
Redis = fast runtime/cache/state
PostgreSQL = durable persistence
```

## **Key Naming**
Document the Redis key namespace convention.

## **Locks**
Explain why Redis locks exist and that they are intended for concurrency/duplicate-operation protection.

---

# **25. Security**
Before completing Phase 3, verify:

- Redis password is never hardcoded.
- Redis credentials are not logged.
- `.env` remains ignored.
- Environment separation exists.
- Test Redis cannot accidentally use production configuration.
- No unsafe deserialization is used.

---

# **26. Definition of Done**
Phase 3 is complete only when:

- Redis dependency added.
- Redis configuration integrated with existing Settings.
- Redis connection manager implemented.
- Connection pooling implemented.
- Redis health check implemented.
- Centralized Redis key builder implemented.
- Generic Redis state manager implemented.
- TTL support implemented.
- Market state abstraction implemented.
- Strategy state abstraction implemented.
- Position state abstraction implemented.
- Distributed lock implemented.
- Duplicate-event protection implemented.
- Serialization handled safely.
- Redis error handling implemented.
- Unit tests implemented.
- Integration test implemented if practical.
- Existing Phase 1 and Phase 2 tests still pass.
- README updated.
- No trading strategy implemented.
- No Kite integration implemented.
- No orders placed.
- No live trading implemented.

---

# **IMPORTANT — STOP AFTER PHASE 3**
Do NOT implement Phase 4.

Do NOT implement:

- Kite Connect
- option-chain discovery
- NIFTY/BANK NIFTY contract selection
- trading strategy
- signal generation
- broker reconciliation
- S3

At the end, report:

1. Redis architecture implemented.
2. Redis key structure.
3. How to start Redis locally.
4. How to verify Redis health.
5. Any architectural decisions made.
6. Any TODOs for Phase 4.

Do not proceed to Phase 4 without my approval.
