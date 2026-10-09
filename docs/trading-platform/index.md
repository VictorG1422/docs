# Algo Trading System

A production-oriented, modular-monolith algorithmic trading system for
**NSE NIFTY / BANK NIFTY options**, built on the **Zerodha Kite Connect** API.

Code repository: [VictorG1422/trading_platform](https://github.com/VictorG1422/trading_platform)

> **This repository currently contains the application framework only.**
> The actual entry/exit trading strategy is **not implemented** -- it is
> explicitly left as `TODO` so it can be plugged in later without
> restructuring the codebase. Phase 9 execution accepts only persisted,
> risk-approved TradeIntents and remains disabled by default; it does not
> make the strategy run or generate trades. Phase 10 reconciles existing
> broker/PostgreSQL/Redis state after the fact -- it never generates a
> signal or places an order either. Phase 11 backtests/paper-trades
> whatever strategy is configured against historical/live data -- it
> never places a real broker order either (see [Phase 11](phases/phase-11-backtesting-paper.md)).

## Documentation map

- [Architecture & Data Flow](architecture.md) -- end-to-end tick/signal/order
  pipeline, instrument/underlying support, and repository layout.
- [Module Reference](module-reference.md) -- a table of every module and its
  responsibility.
- [Setup & Configuration](setup.md) -- installation, every environment
  variable, and the AWS Secrets Manager integration.
- [Operations](operations.md) -- running in paper mode, Docker, PostgreSQL/
  Redis setup, migrations, utility scripts, and logging.
- [Testing](testing.md) -- how to run unit/integration tests.
- [Safety Mechanisms & Roadmap](safety-and-limitations.md) -- built-in safety
  guarantees and what remains to be implemented.
- [Strategy Specification](strategy-specification.md) -- the (currently
  unfilled) template a real trading strategy must be specified against.
- **Phase build log** -- the original phase prompts ([Phase Specs](specs/00-project-initialization.md))
  and the detailed write-up of what was actually built in each phase
  ([Phase Writeups](phases/phase-01-foundation.md)):
  - Phase 1 -- Configuration, Logging & Application Foundation
  - Phase 2 -- PostgreSQL Database Layer
  - Phase 3 -- Redis State & Cache Layer
  - Phase 4 -- Broker Integration & Market Data Foundation
  - Phase 5 -- Instrument Discovery & Option Chain Foundation
  - Phase 6 -- Market Data & Technical Analysis Engine Foundation
  - Phase 7 -- Trading Strategy & Signal Engine
  - Phase 8 -- Risk & Position Management
  - Phase 9 -- Order Execution
  - Phase 10 -- Broker Reconciliation & State Consistency
  - Phase 11 -- Backtesting & Paper Trading Engine
