# Phase 4 — Broker Integration & Market Data Foundation

Phase 1, Phase 2, and Phase 3 have been completed.

PostgreSQL has been created and configured.

Redis has been created and configured.

Phase 3 implemented Redis infrastructure, runtime state management, caching, market-state abstractions, distributed locks, duplicate-event protection, serialization, health checks, and tests.

Now implement PHASE 4 ONLY.

The objective of Phase 4 is to implement the Kite Connect broker integration and market-data foundation.

IMPORTANT:

Do NOT implement trading strategy.

Do NOT implement signal generation.

Do NOT implement option-chain selection.

Do NOT implement NIFTY/BANK NIFTY contract selection.

Do NOT implement order placement.

Do NOT implement stop loss.

Do NOT implement targets.

Do NOT implement position reconciliation.

Do NOT implement risk management.

Do NOT implement live trading.

Do NOT implement Phase 5 or later phases.

This phase is ONLY for broker integration, instrument master data, market-data WebSocket infrastructure, tick processing, and integration with the existing PostgreSQL and Redis architecture.

## 1. Inspect Existing Project First

Before making any changes:

- Inspect the complete repository.
- Review Phase 1 architecture.
- Review Phase 1 configuration.
- Review Phase 1 logging.
- Review Phase 1 health checks.
- Review Phase 2 PostgreSQL implementation.
- Review Phase 3 Redis implementation.
- Review existing database models and migrations.
- Review existing tests.
- Review existing project structure.
- Reuse the existing architecture.
- Do not create duplicate configuration systems.
- Do not create duplicate logging systems.
- Do not rewrite working code unnecessarily.

Before implementation, briefly explain:

1. How Kite Connect will integrate with the existing architecture.
2. How authentication will work.
3. How instrument synchronization will work.
4. How WebSocket market data will flow into Redis.
5. Which files will be created.
6. Which files will be modified.

Then implement Phase 4.

## 2. Kite Connect Dependency

Use the official/mature Python Kite Connect client library.

Add the required dependency using the project's existing dependency-management approach.

Do not introduce unnecessary broker frameworks.

Keep the Kite implementation isolated behind an application-level broker abstraction.

## 3. Kite Configuration

Extend the existing configuration system.

Do NOT create a second configuration system.

Add configuration for:

```
KITE_API_KEY
KITE_API_SECRET
KITE_ACCESS_TOKEN
KITE_REQUEST_TIMEOUT
KITE_WS_TIMEOUT
KITE_RECONNECT_ENABLED
KITE_MAX_RECONNECT_ATTEMPTS
```

If useful, also support:

```
KITE_ENVIRONMENT
```

Use sensible defaults where appropriate.

Do not hardcode:

- API key
- API secret
- Access token
- Credentials

Credentials must come from environment/configuration.

Never log credentials.

Ensure `.env` remains ignored by Git.

## 4. Kite Client Abstraction

Create a clean broker client abstraction.

The rest of the application must not directly depend on the Kite SDK.

Conceptually:

```
Trading Application
->
Broker Interface
->
Kite Broker Client
->
Kite Connect API
```

Create an abstraction that can eventually support operations such as:

```
get_profile()
get_instruments()
get_ltp()
get_quote()
get_ohlc()
```

Only implement operations required by Phase 4.

Do NOT implement order placement.

Do NOT implement trading operations.

Do not tightly couple business logic to Kite SDK classes.

## 5. Kite Authentication

Implement the Kite authentication infrastructure.

Support the normal Kite authentication flow:

```
API key
->
Kite login
->
Request token
->
Access token
->
Authenticated Kite client
```

Do NOT build browser automation.

Do NOT automate login using username/password.

The application should support supplying the required authentication information through configuration or an appropriate controlled authentication workflow.

Handle:

- missing credentials
- invalid credentials
- authentication failure
- expired access token
- API authentication errors

Do not expose secrets in logs or exception messages.

## 6. Kite Health Check

Extend the existing health-check framework.

Add Kite health information.

The health system should be able to distinguish:

```
KITE_CONNECTED
KITE_AUTHENTICATED
KITE_UNAVAILABLE
KITE_AUTH_FAILED
KITE_TOKEN_EXPIRED
```

Example health response:

```
application: healthy
postgresql: healthy
redis: healthy
kite: healthy
```

If Kite is unavailable:

```
kite: unhealthy
```

Do not crash the entire application merely because an external broker health check fails.

Do not expose credentials in health responses.

## 7. Kite Instrument Master Data

Implement Kite instrument downloading.

Flow:

```
Kite
->
Instrument Downloader
->
Validation
->
PostgreSQL
```

Download the instrument master data provided by Kite.

Store the required instrument information in PostgreSQL.

Relevant fields may include:

```
instrument_token
exchange_token
tradingsymbol
name
last_price
expiry
strike
tick_size
lot_size
instrument_type
segment
exchange
```

Use the existing Phase 2 PostgreSQL architecture.

Do not create a second database layer.

## 8. Instrument Repository

Create an instrument repository abstraction.

It should support operations such as:

```
save_instruments()
get_by_token()
get_by_symbol()
get_by_exchange()
get_by_expiry()
get_options()
```

Use PostgreSQL as the authoritative source for instrument information.

Keep repository responsibilities separate from broker communication.

## 9. Instrument Synchronization

Create a controlled instrument synchronization process.

Flow:

```
Application
->
Instrument Sync
->
Kite
->
Validation
->
PostgreSQL
```

Handle:

- duplicate instruments
- changed instruments
- expired instruments
- missing instruments
- malformed records
- database transaction failures

Do not blindly insert duplicate records every time synchronization runs.

Make synchronization safe to run repeatedly.

Use transactions where appropriate.

## 10. Market Data WebSocket

Implement Kite WebSocket infrastructure.

Future flow:

```
Kite WebSocket
->
WebSocket Client
->
Tick Processor
->
MarketStateStore
->
Redis
```

The WebSocket infrastructure should support:

- connection
- authentication
- subscription
- unsubscribe
- reconnection
- connection state
- graceful shutdown

Do not subscribe to any specific trading instruments yet.

Do not hardcode NIFTY, BANK NIFTY, or any particular option contract.

Phase 4 provides the infrastructure only.

## 11. Tick Processor

Create a dedicated tick-processing abstraction.

Flow:

```
Kite Tick
->
Tick Processor
->
Validation
->
Normalization
->
MarketStateStore
->
Redis
```

Normalize incoming market data into the application's internal representation.

Potential fields include:

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

Validate incoming data before storing it.

Use the existing Phase 3 MarketStateStore.

Do not put strategy logic into the tick processor.

## 12. Market State

Use the Phase 3 Redis market-state abstraction.

Do not directly scatter Redis commands throughout the WebSocket implementation.

Use:

```
WebSocket Client
->
Tick Processor
->
MarketStateStore
->
Redis State Manager
```

Support:

```
set_latest_tick()
get_latest_tick()
delete_latest_tick()
```

Use `instrument_token` as the primary lookup identifier.

Do not store historical tick data in Redis.

## 13. Subscription Manager

Create a reusable subscription abstraction.

Support operations such as:

```
subscribe(instrument_tokens)
unsubscribe(instrument_tokens)
```

The subscription manager should be independent of strategy logic.

Do not hardcode:

- NIFTY
- BANK NIFTY
- specific expiry
- specific strike
- specific option contract

Those decisions belong to a later phase.

## 14. Redis Market State Integration

Integrate the Phase 4 tick pipeline with the Phase 3 Redis implementation.

Do not create a second Redis client.

Use the existing:

- Redis Manager
- Redis Connection Pool
- Redis State Manager
- MarketStateStore

Store only the latest/current runtime market state.

Do not use Redis as a historical market-data database.

## 15. Market Data TTL

Use the TTL infrastructure implemented in Phase 3.

Latest market state should not remain valid forever.

Support detection of stale market data.

Clearly distinguish:

- temporary/runtime market state

from:

- durable/historical data

PostgreSQL remains the durable source of truth.

Do not blindly apply TTL to information that must survive recovery.

## 16. Market Data Staleness

Implement a mechanism for detecting stale market data.

Track the timestamp of the latest tick.

The application should be able to determine whether market data is:

- fresh
- stale
- unavailable

Expose this through the appropriate health/status abstraction.

Do not make trading decisions based on market-data staleness in Phase 4.

## 17. WebSocket Reconnection

Implement controlled reconnection behavior.

Expected lifecycle:

```
CONNECTED
->
DISCONNECTED
->
RECONNECTING
->
CONNECTED
->
RESTORE SUBSCRIPTIONS
```

Use bounded retry/backoff.

Do not create an infinite tight reconnect loop.

Support configuration for:

- reconnect enabled/disabled
- maximum reconnect attempts
- connection timeout
- WebSocket timeout

After successful reconnection, restore the active subscriptions where appropriate.

## 18. Kite API Error Handling

Create clean broker-specific exceptions.

Potential hierarchy:

```
BrokerError
AuthenticationError
AuthorizationError
ConnectionError
TimeoutError
RateLimitError
InvalidRequestError
BrokerUnavailableError
```

Map Kite-specific errors into application-level exceptions where practical.

Do not leak secrets through exception messages.

Do not silently swallow errors.

## 19. API Request Safety

Centralize Kite API requests through the broker abstraction.

Handle:

- timeouts
- connection failures
- API errors
- rate limits
- authentication failures

Avoid unnecessary repeated API requests.

Do not introduce complex rate-limiting infrastructure unless the existing application actually requires it.

Keep the implementation simple and maintainable.

## 20. Logging

Use the existing Phase 1 logging system.

Important events may include:

```
KITE_CONNECTED
KITE_DISCONNECTED
KITE_AUTHENTICATED
KITE_AUTH_FAILED
KITE_ERROR
KITE_WS_CONNECTED
KITE_WS_DISCONNECTED
KITE_WS_RECONNECTING
KITE_WS_RECONNECTED
KITE_SUBSCRIPTION_UPDATED
KITE_INSTRUMENT_SYNC_STARTED
KITE_INSTRUMENT_SYNC_COMPLETED
KITE_INSTRUMENT_SYNC_FAILED
```

Do NOT log:

- API secrets
- API keys where unnecessary
- access tokens
- passwords
- sensitive configuration

Do NOT log every market-data tick at INFO level.

High-frequency tick logging would create unnecessary noise and performance problems.

## 21. Performance

Kite WebSocket data will eventually sit on the real-time trading path.

Keep tick processing lightweight.

Avoid:

- unnecessary serialization
- unnecessary Redis round trips
- large Redis values
- blocking operations
- keyspace scans
- storing historical tick data in Redis
- unnecessary database writes for every tick

Do not prematurely optimize with Lua scripts or complex Redis structures unless there is a real requirement.

## 22. PostgreSQL vs Redis Responsibility

Document the architecture clearly.

PostgreSQL is responsible for durable information such as:

- instruments
- orders
- positions
- trades
- signals
- historical records

Redis is responsible for fast runtime information such as:

- latest ticks
- current market state
- temporary runtime state
- locks
- deduplication
- cache
- strategy runtime state

Redis must NOT become the only source of truth.

Kite is responsible for:

- broker API
- instrument source
- market-data source

Future phases may use Kite for order execution, but Phase 4 must NOT implement that.

## 23. Testing

Create unit tests for the Kite and market-data layers.

Configuration tests should cover:

- valid configuration
- missing API key
- missing access token
- invalid timeout
- environment separation

Kite client tests should cover:

- successful initialization
- authentication failure
- timeout
- API error
- broker unavailable

Instrument synchronization tests should cover:

- valid instrument
- duplicate instrument
- malformed instrument
- database failure
- updating existing instrument
- repeated synchronization

WebSocket tests should cover:

- connection
- disconnect
- reconnect
- subscription
- unsubscribe
- subscription restoration after reconnect

Tick processor tests should cover:

- valid tick accepted
- invalid tick rejected
- valid tick stored in Redis
- new tick replacing previous latest tick

Market state tests should cover:

- set latest tick
- get latest tick
- update latest tick
- delete latest tick
- stale market data detection

## 24. Integration Testing

If practical, add integration tests.

Do NOT require developers to accidentally connect to production Kite or production Redis.

Use isolated test configuration.

For WebSocket tests, prefer a mocked/fake WebSocket server rather than relying on live market data.

Integration flow should test something similar to:

```
Kite/mock broker
->
Tick Processor
->
Redis
->
Market State
```

Instrument synchronization should test:

```
Kite/mock broker
->
Instrument Sync
->
PostgreSQL
```

## 25. Security Verification

Before completing Phase 4 verify:

- Kite API key is not hardcoded.
- Kite API secret is not hardcoded.
- Kite access token is not hardcoded.
- Credentials are not logged.
- `.env` remains ignored.
- Test environment cannot accidentally use production credentials.
- No secrets are committed to Git.
- No unsafe deserialization is introduced.
- Authentication tokens are not unnecessarily stored in Redis.
- Error messages do not expose credentials.

## 26. README Update

Update README with:

### Kite Setup

Explain:

- Kite Connect requirements
- required environment variables
- authentication process
- access-token handling
- development configuration

### Instrument Synchronization

Explain:

```
Kite
->
Instrument Sync
->
PostgreSQL
```

### Market Data

Explain:

```
Kite WebSocket
->
Tick Processor
->
Redis MarketStateStore
```

### Redis vs PostgreSQL

Clearly document their responsibilities.

### Development Mode

Explain how to run the application without live trading.

## 27. Important — Do NOT Implement These Features

Do NOT implement:

- Trading strategy.
- Technical indicators.
- AI analysis.
- Signal generation.
- Option-chain discovery logic.
- NIFTY option selection.
- BANK NIFTY option selection.
- Expiry selection logic.
- Strike selection logic.
- Entry logic.
- Exit logic.
- Stop-loss logic.
- Target logic.
- Order placement.
- Position reconciliation.
- Risk engine.
- Broker order management.
- Live trading.
- S3.
- Historical-data pipeline.
- Phase 5.

Only implement Phase 4.

## 28. Definition of Done

Phase 4 is complete only when:

- Kite dependency added.
- Kite configuration integrated with existing Settings.
- Kite broker client abstraction implemented.
- Kite authentication infrastructure implemented.
- Kite health check implemented.
- Instrument downloader implemented.
- Instrument repository implemented.
- Instrument synchronization implemented.
- PostgreSQL instrument storage implemented.
- Kite WebSocket client implemented.
- Tick processor implemented.
- Subscription manager implemented.
- Redis market-state integration implemented.
- WebSocket reconnect mechanism implemented.
- Market-data staleness detection implemented.
- Broker-specific error handling implemented.
- Unit tests implemented.
- Integration tests implemented where practical.
- Phase 1 tests still pass.
- Phase 2 tests still pass.
- Phase 3 tests still pass.
- README updated.
- No trading strategy implemented.
- No signals generated.
- No orders placed.
- No live trading implemented.
- No Phase 5 work implemented.

## 29. Final Report

At the end of implementation, report:

1. Kite architecture implemented.
2. Files created.
3. Files modified.
4. Kite authentication approach.
5. Instrument synchronization architecture.
6. WebSocket architecture.
7. Tick processing flow.
8. Redis market-state integration.
9. Reconnection strategy.
10. Health-check behavior.
11. Tests created and their results.
12. Existing Phase 1, Phase 2, and Phase 3 test results.
13. Any architectural decisions made.
14. Any known limitations.
15. TODOs for Phase 5.

IMPORTANT:

STOP AFTER PHASE 4.

Do not implement Phase 5.

Do not implement trading strategy.

Do not place any orders.

Do not enable live trading.
