# Project Initialization

Pre-phase prompts that set up the project before phase-based development began.

## Query 1 — Algo Trading System — Project Initialization Prompt

I am building a **production-oriented Python algorithmic trading system for NSE NIFTY and BANK NIFTY options using the Zerodha Kite Connect API**.

Your job right now is **NOT to implement the trading strategy itself**. First, create a clean, scalable project structure and the basic application framework so that the strategy can be plugged in later.

## **1. Project Goals**
The system will eventually:

1. Connect to Zerodha Kite Connect.
2. Receive real-time market data through Kite WebSocket.
3. Identify the required NIFTY/BANK NIFTY option contracts.
4. Subscribe only to required instrument tokens rather than the entire option chain.
5. Process real-time ticks.
6. Maintain current and previous market state.
7. Generate trading signals based on a strategy that will be implemented later.
8. Place orders through Kite Connect.
9. Manage Stop Loss (SL) and Target (TGT).
10. Track open positions and orders.
11. Persist important data/state.
12. Recover safely after application restart/crash.
13. Store historical/snapshot data for analysis.
14. Provide structured logging and error handling.
15. Eventually run on AWS.

## **2. Technology Stack**
Use:

- Python 3.12+
- Zerodha Kite Connect API
- PostgreSQL
- Redis
- AWS EC2
- AWS S3
- pytest
- Docker support should be possible later, but do not over-engineer it now.

Preferred architecture:

- EC2 → main application
- PostgreSQL/RDS → persistent data
- Redis/ElastiCache → fast temporary state/cache
- S3 → option-chain snapshots and larger historical files

Do NOT introduce Kubernetes, EKS, ECS, Kafka, Celery, or other distributed infrastructure at this stage.

Start with a modular monolith.

The architecture should allow individual components to be separated into services later if the system grows.

---

# **3. Repository Structure**
Create a clean structure similar to:

```
algo-trading/
│
├── README.md
├── .gitignore
├── .env.example
├── requirements.txt
├── pyproject.toml
├── Dockerfile
│
├── config/
│   └── settings.py
│
├── src/
│   └── trading_system/
│       │
│       ├── __init__.py
│       ├── main.py
│       │
│       ├── broker/
│       │   ├── __init__.py
│       │   ├── kite_client.py
│       │   ├── websocket.py
│       │   └── orders.py
│       │
│       ├── market_data/
│       │   ├── __init__.py
│       │   ├── tick_processor.py
│       │   ├── instrument_manager.py
│       │   └── option_chain.py
│       │
│       ├── strategy/
│       │   ├── __init__.py
│       │   ├── base.py
│       │   └── strategy.py
│       │
│       ├── execution/
│       │   ├── __init__.py
│       │   ├── order_manager.py
│       │   └── position_manager.py
│       │
│       ├── risk/
│       │   ├── __init__.py
│       │   └── risk_manager.py
│       │
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── postgres.py
│       │   ├── redis.py
│       │   └── s3.py
│       │
│       ├── models/
│       │   ├── __init__.py
│       │   ├── tick.py
│       │   ├── instrument.py
│       │   ├── order.py
│       │   ├── position.py
│       │   └── signal.py
│       │
│       ├── utils/
│       │   ├── __init__.py
│       │   ├── logger.py
│       │   └── time_utils.py
│       │
│       └── exceptions/
│           └── __init__.py
│
├── tests/
│   ├── unit/
│   └── integration/
│
├── scripts/
│   ├── download_instruments.py
│   └── health_check.py
│
├── data/
│   └── .gitkeep
│
└── logs/
    └── .gitkeep
```

You may improve this structure if there is a strong architectural reason, but keep it simple and modular.

---

# **4. Configuration**
Use environment variables for secrets and environment-specific configuration.

Create `.env.example` containing placeholders such as:

```
KITE_API_KEY=
KITE_API_SECRET=
KITE_ACCESS_TOKEN=

POSTGRES_HOST=
POSTGRES_PORT=
POSTGRES_DATABASE=
POSTGRES_USER=
POSTGRES_PASSWORD=

REDIS_HOST=
REDIS_PORT=
REDIS_PASSWORD=

AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=

TRADING_MODE=paper
LOG_LEVEL=INFO
```

Never hardcode credentials.

Never commit `.env`.

Add `.env` to `.gitignore`.

---

# **5. Trading Modes**
The application must support at least:

- `paper`
- `live`

Default should be:

`paper`

The initial implementation must **not accidentally place live orders**.

Create an execution abstraction so that:

```
Strategy
   ↓
Signal
   ↓
Order Manager
   ↓
Execution Interface
      ├── Paper Execution
      └── Kite Live Execution
```

Live execution should require an explicit configuration such as:

`TRADING_MODE=live`

Do not make live trading the default.

---

# **6. Broker Layer**
Create a broker abstraction rather than coupling the entire application directly to Kite.

Example conceptual interface:

```
class Broker:
    def connect():
        ...

    def get_instruments():
        ...

    def subscribe():
        ...

    def place_order():
        ...

    def modify_order():
        ...

    def cancel_order():
        ...

    def get_positions():
        ...

    def get_orders():
        ...
```

Implement Kite-specific functionality inside the broker layer.

The rest of the application should interact with the broker through abstractions/interfaces wherever practical.

---

# **7. Market Data**
Create a WebSocket component responsible for:

- Connecting to Kite WebSocket.
- Authenticating.
- Subscribing to instrument tokens.
- Receiving ticks.
- Handling reconnects.
- Handling connection failures.
- Passing ticks to the tick processor.

Do NOT put strategy logic inside the WebSocket callback.

Expected flow:

```
Kite WebSocket
      ↓
WebSocket Handler
      ↓
Tick Processor
      ↓
Market State
      ↓
Strategy Engine
```

The tick processor should normalize incoming broker data into an internal `Tick` model.

Example fields:

- instrument_token
- exchange
- tradingsymbol
- timestamp
- last_price
- last_quantity
- volume
- open
- high
- low
- close
- oi
- bid/ask if available

Keep the model extensible.

---

# **8. Instrument Management**
The system needs to manage NSE instruments and option contracts.

Create an instrument manager capable of:

- Downloading instrument information.
- Updating instrument information.
- Identifying NIFTY contracts.
- Identifying BANK NIFTY contracts.
- Finding expiry.
- Finding strike price.
- Identifying CE/PE.
- Mapping trading symbols to instrument tokens.

Instrument data should eventually be stored in PostgreSQL.

Do not assume a fixed instrument token.

Do not hardcode option symbols or expiry dates.

---

# **9. Option Chain**
Create an option-chain abstraction.

It should eventually support:

```
Underlying
Expiry
Strike
CE
PE
Instrument Token
LTP
OI
Volume
Bid
Ask
```

The option-chain component should be independent of the strategy.

The strategy will later decide whether it needs:

- ATM
- ITM
- OTM
- CE
- PE
- multiple strikes

Do NOT hardcode the final strike-selection strategy yet.

---

# **10. Strategy Architecture**
Create a strategy interface/base class.

Example:

```
class Strategy:
    def on_tick(self, tick):
        pass

    def on_market_data(self, market_data):
        pass

    def generate_signal(self):
        pass
```

Create a placeholder strategy implementation.

It should return something like:

```
BUY
SELL
EXIT
HOLD
```

or a structured `Signal` model.

The strategy must NOT directly place orders.

Expected flow:

```
Market Data
     ↓
Strategy
     ↓
Signal
     ↓
Risk Manager
     ↓
Order Manager
     ↓
Broker
```

Keep the actual entry/exit rules as TODOs.

Do not invent a strategy.

---

# **11. Risk Management**
Create a dedicated risk manager.

Eventually it should handle:

- Maximum daily loss.
- Maximum position size.
- Maximum number of trades.
- Stop Loss.
- Target.
- Position exposure.
- Duplicate order prevention.
- Trading-session validation.
- Emergency shutdown.

For now, implement the framework/interfaces and safe defaults.

---

# **12. Order Management**
Create an order manager responsible for:

- Creating orders.
- Tracking order status.
- Preventing duplicate orders.
- Handling rejected orders.
- Handling partial fills.
- Managing SL.
- Managing target.
- Updating position state.

Do not allow the strategy to call Kite's order API directly.

Example:

```
Strategy
   ↓
Signal
   ↓
Risk Manager
   ↓
Order Manager
   ↓
Broker
```

---

# **13. Position Management**
Maintain an internal representation of:

- Open positions.
- Entry price.
- Quantity.
- Direction.
- Stop Loss.
- Target.
- Current P&L.
- Realized P&L.
- Unrealized P&L.
- Entry timestamp.
- Exit timestamp.

The system should eventually be able to rebuild its state after a restart by querying the broker and database.

---

# **14. PostgreSQL**
Use PostgreSQL for persistent information.

Potential tables/entities:

```
instruments
ticks
orders
positions
trades
signals
strategy_runs
```

Do not write every high-frequency tick blindly into PostgreSQL.

The architecture should allow high-frequency data to be processed in memory/Redis and only required data to be persisted.

Use PostgreSQL for durable business state and important historical records.

Use migrations rather than manually creating tables.

---

# **15. Redis**
Use Redis for fast-changing state such as:

- Current market state.
- Latest tick.
- Current option-chain state.
- Active positions.
- Strategy state.
- Temporary locks.
- Duplicate-order protection.

Do not treat Redis as the permanent source of truth.

PostgreSQL/broker should be used for recovery and durable state where appropriate.

---

# **16. S3**
Use S3 for larger historical/snapshot data such as:

- Option-chain snapshots.
- Raw market-data files.
- Backtesting datasets.
- Daily archives.

Do not make S3 a dependency for every individual tick.

---

# **17. Trading Session**
The application should understand Indian market timings.

Create utilities for:

- Market open.
- Market close.
- Trading day.
- Expiry day.
- Holidays.

Do not hardcode market-time logic throughout the application.

Centralize it in `time_utils` / market-calendar functionality.

---

# **18. Logging**
Implement structured application logging.

Logs should clearly identify:

- Timestamp.
- Component.
- Log level.
- Event.
- Instrument.
- Order ID where applicable.
- Strategy signal where applicable.

Important events:

```
APPLICATION_STARTED
BROKER_CONNECTED
WEBSOCKET_CONNECTED
SUBSCRIPTION_UPDATED
TICK_RECEIVED
SIGNAL_GENERATED
ORDER_CREATED
ORDER_FILLED
ORDER_REJECTED
POSITION_OPENED
POSITION_CLOSED
RISK_LIMIT_TRIGGERED
WEBSOCKET_RECONNECTED
APPLICATION_SHUTDOWN
```

Never log API secrets or sensitive credentials.

---

# **19. Error Handling**
Implement clean exception handling.

The application should handle:

- Kite API errors.
- WebSocket disconnects.
- Database connection failures.
- Redis failures.
- Invalid instruments.
- Order rejection.
- Network failures.
- Unexpected application exceptions.

Do not silently swallow exceptions.

Use retries only where appropriate.

Avoid infinite retry loops.

---

# **20. Testing**
Create pytest structure for:

- Instrument selection.
- Option-chain logic.
- Tick processing.
- Signal generation.
- Risk management.
- Order management.
- Position management.
- Time/session utilities.

Initially create basic tests for the framework.

Do NOT connect to the real Kite API during unit tests.

Use mocks/fakes.

---

# **21. Main Application**
Create a clean application entry point.

Conceptually:

```
main.py
   ↓
Load Configuration
   ↓
Initialize Logger
   ↓
Initialize PostgreSQL
   ↓
Initialize Redis
   ↓
Initialize Broker
   ↓
Load Instruments
   ↓
Initialize Market Data
   ↓
Initialize Strategy
   ↓
Initialize Risk Manager
   ↓
Initialize Order Manager
   ↓
Start Trading Engine
```

Create a `TradingEngine` orchestration layer if useful.

The engine should coordinate components without containing all their implementation details.

---

# **22. Important Safety Requirements**
This is a real-money trading project.

Therefore:

1. Default mode MUST be paper trading.
2. Never hardcode API credentials.
3. Never automatically enable live trading.
4. Never place an order from a unit test.
5. Never let the strategy directly call the broker order API.
6. Add clear logging around every order.
7. Add duplicate-order protection.
8. Add basic risk checks before order submission.
9. Handle application restart safely.
10. Fail safely when critical dependencies are unavailable.

---

# **23. Development Philosophy**
Keep the first version simple.

Do NOT:

- Build microservices.
- Introduce Kubernetes.
- Introduce Kafka.
- Build unnecessary abstractions.
- Create hundreds of files.
- Implement a fake complex trading strategy.
- Add unnecessary frameworks.
- Over-engineer the system.

Prioritize:

```
Correctness
↓
Safety
↓
Testability
↓
Observability
↓
Performance
↓
Scalability
```

The system should be modular enough that components can later be extracted into separate services if required.

---

# **24. Initial Deliverable**
For this task, create the initial repository with:

1. Complete folder structure.
2. Python package structure.
3. Configuration management.
4. `.env.example`.
5. `.gitignore`.
6. `README.md`.
7. Requirements/dependency configuration.
8. Logging framework.
9. Basic PostgreSQL abstraction.
10. Basic Redis abstraction.
11. Basic S3 abstraction.
12. Kite broker abstraction.
13. WebSocket abstraction.
14. Instrument manager skeleton.
15. Tick model.
16. Signal model.
17. Order model.
18. Position model.
19. Strategy interface.
20. Risk manager skeleton.
21. Order manager skeleton.
22. Position manager skeleton.
23. Trading engine skeleton.
24. Paper-trading execution implementation.
25. Basic pytest tests.
26. `main.py` that can start the application safely in paper mode.

The application should be runnable after installation with a simple command.

At this stage, **do not implement the actual trading strategy or live order execution logic**.

Use clear TODO markers wherever strategy-specific rules will later be inserted.

Before creating files, briefly explain the architecture you are going to implement. Then create the files.

After creating them, provide:

- Final folder tree.
- How to install dependencies.
- How to configure `.env`.
- How to run in paper mode.
- How to run tests.
- What remains to be implemented next.

Make the code production-quality, readable, strongly typed where practical, and easy for another developer to understand.

---

## Query 2 — README Update Request

update all the things how to run the code how the code workd all details should be in readme file update that
