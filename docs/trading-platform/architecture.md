## Which instruments/underlyings does this support?

**Nothing in this codebase is hardcoded to NIFTY/BANKNIFTY.** They are
used throughout as the *running examples* (they're the most liquid NSE
F&O underlyings and what the docs' illustrations use), but every
layer is driven entirely by whatever instrument data you actually
synchronize from Kite (`InstrumentSyncService`/`sync_instruments()`,
Phase 4) into PostgreSQL -- never by a fixed list baked into the code:

- **Underlying discovery** (`UnderlyingDiscoveryService.is_supported()`/
  `discover()`, Phase 5) checks the *real synchronized data* -- there is
  no hardcoded allowlist. Any underlying with synced F&O instruments
  (NIFTY, BANKNIFTY, FINNIFTY, MIDCPNIFTY, or single-stock F&O like
  RELIANCE/TCS/INFY/...) is automatically "supported".
- **Expiries** (`ExpiryService`) read the actual synced expiry dates --
  never assume a weekly cycle. This matters because most single-stock
  F&O only has *monthly* expiries (no weekly), unlike NIFTY/BANKNIFTY.
- **Strikes** (`OptionChainService`, `OptionSelector`) are read from the
  actual synced strike ladder for that underlying+expiry, and ATM/OTM/ITM
  offsets are applied as list-index steps into that real ladder -- never
  a hardcoded price step. This matters because strike spacing varies
  enormously across underlyings (NIFTY: 50, BANKNIFTY: 100, many
  lower-priced stocks: 2.5/5/10/20).
- **Lot size** (`Instrument.lot_size`, used by `PositionSizer`) always
  comes from the synced instrument row -- never a hardcoded NIFTY/
  BANKNIFTY lot size. Lot sizes vary drastically per stock.
- **Patterns/indicators/scoring** (`patterns.py`/`scoring.py`) operate on
  generic OHLCV `Candle` data -- nothing about them is index-specific.
  `oi_buildup` works identically for index or single-stock F&O (both
  report open interest).
- **Risk engine/position sizing** (`RiskPolicy`/`RiskEngine`/
  `PositionSizer`) are entirely config- and instrument-driven -- no
  underlying-specific branches anywhere.

**What you'd actually need to do to trade a new underlying/stock**:
sync its F&O instruments (`sync_instruments()`), construct a
`StrategyConfig` with that underlying's own `instrument_token` (its
futures contract is usually the best token to evaluate candles against),
and construct/`start()` a `StrategyEngine` (+ optionally an
`OptionSelector`/`RiskEngine`) for it -- exactly the same steps as for
NIFTY/BANKNIFTY. Running multiple underlyings simultaneously just means
one `StrategyEngine` instance per underlying+timeframe (the framework
was always designed this way; nothing is a process-wide singleton tied
to one underlying).

**Known gaps if you extend to less-liquid single-stock F&O** (these
don't affect NIFTY/BANKNIFTY today, but matter more for illiquid names):
option bid/ask-spread and open-interest/volume liquidity thresholds are
not yet implemented anywhere (`models/tick.py`'s `bid`/`ask` fields exist
but are unpopulated/unused) -- see
[Safety Mechanisms & Roadmap](safety-and-limitations.md#what-remains-to-be-implemented).
Trade responsibly and validate liquidity yourself before extending
beyond the most liquid names.

## Architecture & data flow

```
Kite WebSocket
      │
      ▼
WebSocket Handler (broker/websocket.py)
      │  raw ticks
      ▼
Tick Processor (market_data/tick_processor.py)  ──▶  Market State
      │  normalized Tick                                   │
      ▼                                                    ▼
Strategy (strategy/base.py)  ◀───────────────── Option Chain / Instruments
      │  Signal
      ▼
Risk Manager (risk/risk_manager.py)
      │  approved Signal
      ▼
Order Manager (execution/order_manager.py)
      │  Order
      ▼
Execution Handler ──┬── Paper Execution (default, simulated fills)
                     └── Live Execution (Kite Connect, TRADING_MODE=live only)
      │
      ▼
Position Manager (execution/position_manager.py)
```

Step by step, a tick's journey through the system:

1. `KiteWebSocketClient` (`broker/websocket.py`) connects to Kite's
   streaming API and receives raw tick payloads for subscribed
   instrument tokens.
2. Raw ticks are converted into the normalized `Tick` model
   (`models/tick.py`) and handed to `TickProcessor.process()`.
3. `TickProcessor` updates the thread-safe `MarketState`
   (current + previous tick per instrument) and fans the tick out to
   every registered listener, isolating exceptions so one bad listener
   never breaks the pipeline.
4. The `Strategy` (currently `PlaceholderStrategy`) receives the tick via
   `on_tick`/`on_market_data` and may call `generate_signal()`, returning
   a `Signal` (`models/signal.py`) or `None`. Strategies never touch the
   broker or execution layer.
5. `RiskManager.validate_signal()` checks the signal against daily-loss,
   trade-count, duplicate-signal, and open-position-size limits before
   it is allowed to become an order.
6. The legacy `OrderManager.submit_signal()` path is paper-only and kept
  for compatibility; its live handler now fails closed.
7. Phase 9 `OrderExecutionEngine.execute(intent)` validates a persisted,
  risk-approved `TradeIntent` and calls the Kite broker only through the
  explicit gated `KiteBrokerAdapter` path.
8. Broker acknowledgement is recorded as `SUBMITTED`, never `FILLED`.
  `refresh_status(intent_id)` records confirmed fills and updates the
  Redis position lifecycle only after a fill is reported.

Key design decisions:

- **Broker abstraction** (`broker/base.py::Broker`): the rest of the app
  never imports `kiteconnect` directly outside the `broker/` package.
- **Execution boundary**: the legacy `OrderManager` is paper-only.
  Phase 9's `OrderExecutionEngine` is the only supported live-order route;
  it requires a persisted approved intent and explicit environment gates.
- **Strategy interface** (`strategy/base.py::Strategy`): observes ticks
  and market state, returns `Signal | None`. Contains no order-placement
  logic.
- **Storage split**: PostgreSQL for durable business state (orders,
  positions, trades, signals, instruments), Redis for fast-changing
  state (current ticks, locks, duplicate-order protection), S3 for bulk
  snapshots/archives. High-frequency ticks are **not** written to
  PostgreSQL.
- **TradingEngine** (`engine.py`): orchestrates startup/shutdown of all
  components; contains no business logic itself.

## Repository layout

```
trading_platform/
├── README.md
├── .gitignore
├── .env.example
├── requirements.txt            # runtime deps (used by Dockerfile)
├── requirements-dev.txt        # + pytest/ruff for local development
├── pyproject.toml              # canonical package + tool configuration
├── Dockerfile
├── alembic.ini                 # migrations config
├── migrations/                 # Alembic migrations (schema evolution)
│   ├── env.py
│   ├── script.py.mako
│   └── versions/0001_initial_schema.py
├── config/
│   ├── __init__.py
│   └── settings.py             # env-driven Settings (pydantic-settings)
├── src/trading_system/
│   ├── main.py                 # application entry point
│   ├── engine.py                # TradingEngine orchestration
│   ├── broker/                  # Kite abstraction (REST + WebSocket)
│   ├── market_data/             # tick processing, instruments, option chain
│   ├── strategy/                # Strategy interface + placeholder
│   ├── execution/               # paper/live execution, order & position managers
│   ├── risk/                    # risk manager
│   ├── storage/                 # PostgreSQL / Redis / S3 clients
│   │   ├── postgres.py
│   │   ├── redis/                # Phase 3: pooled client, key builder, state/lock/dedup/cache stores
│   │   ├── s3.py
│   │   └── db/                   # Phase 2: ORM models, repositories
│   ├── models/                  # Tick, Instrument, Signal, Order, Position
│   ├── utils/                   # structured logging, market-time utilities
│   └── exceptions/               # typed exception hierarchy
├── scripts/
│   ├── download_instruments.py
│   └── health_check.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── data/                        # local scratch data (gitignored contents)
└── logs/                        # local logs (gitignored contents)
```
