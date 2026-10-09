# Phase 2 — PostgreSQL Database Layer

Phase 1 has been completed and tested successfully.

Now implement **PHASE 2 only: PostgreSQL database infrastructure and persistence layer**.

Do not implement Redis, Kite integration, WebSocket, trading strategy, order execution, live trading, or S3 in this phase.

---

## **1. First Inspect Phase 1**
Before making changes:

- Inspect the complete existing repository.
- Review the Phase 1 configuration system.
- Review logging implementation.
- Review existing exception handling.
- Review existing tests.
- Reuse the existing architecture.
- Do not unnecessarily modify working Phase 1 code.

Briefly explain how Phase 2 will integrate with the existing Phase 1 architecture.

---

# **2. PostgreSQL Technology**
Use PostgreSQL as the persistent database.

Choose a mature Python PostgreSQL stack appropriate for a production application.

Prefer:

- SQLAlchemy 2.x
- PostgreSQL driver such as psycopg
- Alembic for database migrations

Use asynchronous database access only if there is a clear architectural reason. Do not introduce unnecessary complexity.

---

# **3. Configuration Integration**
Use the existing Phase 1 configuration system.

Add PostgreSQL configuration:

```
POSTGRES_HOST
POSTGRES_PORT
POSTGRES_DATABASE
POSTGRES_USER
POSTGRES_PASSWORD
```

Do not create a second configuration mechanism.

Do not hardcode database credentials.

Update `.env.example`.

---

# **4. Database Module**
Implement a clean database layer.

Conceptually:

```
Application
    ↓
Database Manager
    ↓
SQLAlchemy Engine
    ↓
Session
    ↓
PostgreSQL
```

The rest of the application should not need to know the PostgreSQL connection details.

Create appropriate modules under the existing:

```
storage/
models/
```
structure.

---

# **5. Database Connection**
Implement:

- PostgreSQL engine creation
- Connection pooling
- Session management
- Connection cleanup
- Configuration-driven connection URL

The connection pool should use sensible defaults.

Do not aggressively tune connection-pool parameters without a reason.

---

# **6. Database Health Check**
Extend the Phase 1 health-check framework.

Add:

```
PostgreSQL health check
```

It should verify that the database is reachable.

Expected conceptual result:

```
Application: HEALTHY
PostgreSQL: HEALTHY
```

If PostgreSQL is unavailable, the health check should clearly report it.

Do not expose credentials in the health-check result.

---

# **7. Database Models**
Create SQLAlchemy models for the following core entities.

## **Instrument**
Fields should include approximately:

```
id
instrument_token
exchange
tradingsymbol
name
segment
instrument_type
expiry
strike
lot_size
tick_size
created_at
updated_at
```

Requirements:

- `instrument_token` should be indexed/unique as appropriate.
- Trading symbol should be indexed.
- Expiry should be indexed.
- Strike should be indexed where useful.
- Avoid unnecessary indexes.

The model must support both:

```
NIFTY
BANKNIFTY
```
without hardcoding either one into the database model.

---

# **8. Order Model**
Create an order model capable of tracking both paper and future live orders.

Include concepts such as:

```
id
broker_order_id
instrument_id
side
order_type
quantity
price
trigger_price
status
product
validity
strategy_name
signal_id
created_at
updated_at
filled_at
rejection_reason
metadata
```

Important:

The database model must NOT assume that every order is immediately filled.

Support states such as:

```
PENDING
OPEN
PARTIALLY_FILLED
FILLED
CANCELLED
REJECTED
FAILED
```

Use enums where appropriate.

---

# **9. Position Model**
Create a position model.

Track:

```
id
instrument_id
quantity
average_entry_price
average_exit_price
realized_pnl
unrealized_pnl
status
entry_time
exit_time
strategy_name
created_at
updated_at
```

Possible status:

```
OPEN
CLOSED
```

The model must eventually support reconciliation with the broker.

Do not assume the database is always correct.

---

# **10. Trade Model**
Create a separate `Trade` entity.

A trade represents a completed trading transaction rather than an individual broker order.

Track concepts such as:

```
id
position_id
strategy_name
instrument_id
entry_price
exit_price
quantity
gross_pnl
charges
net_pnl
entry_time
exit_time
exit_reason
created_at
```

The design should allow multiple orders to contribute to a single trade if partial fills are later supported.

---

# **11. Signal Model**
Create a signal model.

Track:

```
id
strategy_name
instrument_id
signal_type
signal_timestamp
signal_price
quantity
stop_loss
target
metadata
created_at
```

Possible signal types:

```
BUY
SELL
EXIT
HOLD
```

Do not implement the actual strategy.

This is only the persistence model.

---

# **12. Relationships**
Create appropriate relationships between:

```
Instrument
    ↓
Order

Instrument
    ↓
Position

Position
    ↓
Trade

Instrument
    ↓
Signal

Signal
    ↓
Order
```

Use foreign keys.

Avoid circular relationships unless genuinely required.

Make relationships easy to query.

---

# **13. IDs**
Use appropriate primary keys.

Prefer a consistent ID strategy throughout the database.

Do not mix random ID formats unnecessarily.

Broker-generated IDs such as Kite order IDs must remain separate from our internal database IDs.

Example:

```
internal order ID
        +
broker order ID
```

This will be important later for reconciliation.

---

# **14. Timestamps**
Use timezone-aware timestamps.

The trading system operates in Indian market time, but database timestamps should be handled consistently.

Prefer storing timestamps in UTC internally while converting to IST for display/business logic where appropriate.

Do not mix naive and timezone-aware datetime objects.

Create reusable timestamp utilities if necessary.

---

# **15. Database Constraints**
Add sensible database constraints.

Examples:

- instrument token uniqueness where appropriate
- non-negative quantities where appropriate
- valid enum values
- required timestamps
- required foreign keys
- appropriate NOT NULL constraints

Do not add overly restrictive constraints that could prevent legitimate states such as:

- partially filled orders
- rejected orders
- cancelled orders

---

# **16. Indexing**
Think carefully about indexes.

Expected frequently queried fields include:

### **Instruments**

```
instrument_token
tradingsymbol
expiry
strike
```

### **Orders**

```
broker_order_id
status
created_at
instrument_id
```

### **Positions**

```
status
instrument_id
```

### **Trades**

```
strategy_name
entry_time
exit_time
```

### **Signals**

```
strategy_name
signal_timestamp
instrument_id
```

Do not create indexes for every column.

---

# **17. Alembic Migrations**
Set up Alembic properly.

Create an initial migration containing all Phase 2 tables.

The project should support:

```
alembic upgrade head
```
and:

```
alembic downgrade
```

Do not manually create production tables from Python code.

Database schema changes should be handled through migrations.

---

# **18. Database Initialization**
Create a clean mechanism for initializing the database connection.

Do NOT automatically destroy/recreate tables when the application starts.

Never do anything equivalent to:

```
DROP TABLE
```
during normal application startup.

The application should assume migrations manage schema creation.

---

# **19. Repository / Data Access Layer**
Avoid putting raw SQL queries throughout the application.

Create a clean persistence/data-access layer.

For example:

```
InstrumentRepository
OrderRepository
PositionRepository
TradeRepository
SignalRepository
```

The exact implementation can follow the existing architecture.

Repositories should provide basic operations such as:

```
create
get_by_id
update
delete where appropriate
list
```

For trading entities, prefer explicit domain operations over unrestricted CRUD where appropriate.

---

# **20. Transaction Management**
Implement proper transaction handling.

Important requirements:

- Commit successful operations.
- Roll back failed transactions.
- Do not leave sessions open.
- Avoid committing partially completed business operations.
- Keep transaction boundaries clear.

For example, when appropriate:

```
Create Order
    ↓
Commit
```

If something fails:

```
Rollback
```

---

# **21. Session Management**
Create safe session handling.

The application should not create one global database session and keep it open forever.

Use controlled session lifecycle.

Conceptually:

```
Request/Operation
    ↓
Create session
    ↓
Perform database operation
    ↓
Commit/Rollback
    ↓
Close session
```

Adapt this to the application's eventual long-running architecture.

---

# **22. Tick Data**
Do NOT create a naive architecture that writes every WebSocket tick directly to PostgreSQL.

The future system will receive high-frequency market data.

For Phase 2:

- Do not implement tick ingestion.
- Do not implement WebSocket.
- Do not create a high-frequency tick persistence pipeline.

A future architecture will use Redis/in-memory processing and selectively persist data.

---

# **23. Testing**
Create database tests.

Use an isolated test database or appropriate test strategy.

Test:

### **Connection**

- valid connection
- invalid configuration
- database unavailable

### **Models**

- Instrument creation
- Order creation
- Position creation
- Trade creation
- Signal creation

### **Relationships**
Verify:

```
Instrument → Orders
Instrument → Positions
Position → Trades
Instrument → Signals
Signal → Orders
```

### **Constraints**
Test important constraints.

### **Transactions**
Test:

- commit
- rollback

Do NOT connect tests to a production database.

Do NOT execute real trades.

---

# **24. Phase 1 Compatibility**
Ensure all existing Phase 1 tests continue to pass.

Run:

```
pytest
```
and verify the complete test suite.

Do not break the existing configuration or logging behavior.

---

# **25. README**
Update README with:

## **PostgreSQL Setup**
Explain:

- PostgreSQL requirement
- required environment variables
- how to create the database
- how to run migrations
- how to verify connectivity

Example workflow:

```
Configure .env
    ↓
Create PostgreSQL database
    ↓
Run migrations
    ↓
Start application
```

Do not include real credentials.

---

# **26. Security**
Before completing Phase 2, verify:

- No database passwords are hardcoded.
- No credentials appear in logs.
- `.env` remains ignored.
- No production database credentials are used in tests.
- SQL injection risks are avoided through SQLAlchemy parameterization.
- No destructive database operation runs automatically at startup.

---

# **27. Definition of Done**
Phase 2 is complete only when:

- PostgreSQL dependency added.
- Database configuration integrated with Phase 1.
- SQLAlchemy configured.
- PostgreSQL connection works.
- Database health check works.
- Instrument model implemented.
- Order model implemented.
- Position model implemented.
- Trade model implemented.
- Signal model implemented.
- Relationships implemented.
- Appropriate indexes added.
- Constraints added.
- Alembic configured.
- Initial migration created.
- Repository/data-access layer implemented.
- Transaction handling implemented.
- Database tests implemented.
- Existing Phase 1 tests still pass.
- README updated.
- No real trading functionality added.

---

# **IMPORTANT — DO NOT IMPLEMENT PHASE 3**
Do NOT implement:

- Redis
- Kite API
- Kite authentication
- Kite WebSocket
- instrument downloading from Kite
- option-chain processing
- strategy logic
- signals generation
- order placement
- paper execution
- live execution
- SL/Target
- position reconciliation
- S3

Stop after Phase 2.

At the end, report:

1. Database schema created.
2. Migration created.
3. Tests added.
4. Full test result.
5. How to run PostgreSQL locally.
6. How to run migrations.
7. Any design decisions or TODOs for Phase 3.

Do not proceed to Phase 3 without my approval.
