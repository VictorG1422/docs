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

!!! tip "Looking for the step-by-step pipeline?"
    See [Flow & Worked Examples](flow-and-examples.md) for a diagram and
    four concrete walkthroughs of how a price tick becomes a trade. This
    section covers the *design decisions* behind that pipeline instead of
    repeating the steps.

**The configured application runtime is the canonical trading path.**
`Application` selects one database-configured mode and wires a `BaseStrategy`
through `StrategyEngine`, `RiskEngine`, and `OrderExecutionEngine`. The
strategy engine emits validated, persisted signals; the runtime hands those
signals to risk and execution. The old Phase 1 `Strategy`/`RiskManager`/
`OrderManager` objects remain in the repository as legacy components; they
are not the configured paper/live/backtest pipeline.

The current selectable strategy is `technical_scoring`: a configurable
multi-timeframe pattern/indicator confluence scorer. `no_op` is also
selectable for deliberately disabling signals. Neither the implementation
nor a passing test suite is a claim that a trading strategy is approved for
production; see [Safety Mechanisms & Roadmap](safety-and-limitations.md).

The dashboard's Redis-backed order-entry switch is shared by configured
backtest, paper, and live execution. Pausing blocks entry signals/orders at
both runtime and final execution checks. Protective exits and
dashboard-admin-authenticated manual exits remain available. Live
submission still requires the separate environment, mode, credential, and
`ORDER_EXECUTION_ENABLED` safeguards.

Risk-approved index CE/PE entries with distinct signals can create separate
entry lots for one contract; same-signal repeats remain deduplicated and
open EQ duplicates remain blocked. Live lots are persisted separately and
reconciled against the broker's net instrument exposure. Paper/backtest
portfolios also keep per-entry lots, so each lot retains its own protective
levels and manual-exit selector.

Key design decisions behind the real pipeline:

- **Broker abstraction** (`broker/base.py::Broker`): the rest of the app
  never imports `kiteconnect` directly outside the `broker/` package.
  Swapping brokers later would mean writing one new adapter, not
  rewriting the strategy/risk/execution layers.
- **A strategy only ever returns an opinion, never places an order**
  (`strategy/base.py::BaseStrategy`): `evaluate()` takes a read-only
  snapshot of the market and returns `BUY`, `SELL`, or `None`. It has no
  access to the broker, the database, or Redis -- which makes a strategy
  trivial to unit-test and impossible to accidentally wire into live
  └── logs/                        # local scratch data (gitignored contents)
  ```
  order requires a persisted, risk-approved `TradeIntent`
  *and* the independent gates `ORDER_EXECUTION_ENABLED=true`,
  `APP_ENV=production`, and the database-selected `APPLICATION_MODE=live`.
  Missing any gate prevents real order submission.
- **Storage is split by how fast and how durable the data needs to be**:
  PostgreSQL holds everything that must survive a restart and be
  auditable (orders, positions, trades, signals, instruments); Redis
  holds everything that's read constantly but can be safely rebuilt
  (latest ticks, locks, duplicate-signal guards); S3 is for bulk
  snapshots. Raw high-frequency ticks are deliberately **not** written to
  PostgreSQL -- only completed candles are.
- **`TradingEngine` (`engine.py`) is a wiring layer, not a decision
  maker**: it constructs every component and exposes explicit methods
  like `execute_trade_intent()`, but nothing in it decides *when* to call
  them -- that stays an explicit, deliberate choice made by whatever
  wires the engine up (see
  [Safety Mechanisms & Roadmap](safety-and-limitations.md#what-remains-to-be-implemented)).


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
│   ├── api/                      # Phase 11 addendum: dashboard and protected settings API
│   │   ├── app.py                 # app factory, renders frontend/templates/, includes routers
│   │   ├── dependencies.py        # Postgres/Redis/Kite singletons + DB session
│   │   ├── serialization.py       # generic ORM row -> JSON dict
│   │   ├── queries.py             # generic "list recent" + instrument enrichment
│   │   └── routers/               # health, market, trading, risk, reconciliation, backtests
│   ├── utils/                   # structured logging, market-time utilities
│   └── exceptions/               # typed exception hierarchy
├── frontend/                    # Phase 11 addendum: dashboard (Python/Jinja2 + jQuery)
│   ├── templates/index.html      # server-rendered by trading_system.api.app
│   └── static/
│       ├── css/style.css
│       └── js/app.js
├── scripts/
│   ├── download_instruments.py
│   ├── health_check.py
│   └── run_dashboard.py          # compatibility shim to the unified entry point
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── data/                        # local scratch data (gitignored contents)
└── logs/                        # local logs (gitignored contents)
```

Generated data and logs are gitignored.
