# Algo Trading System

A production-oriented, modular-monolith algorithmic trading system for
**NSE NIFTY / BANK NIFTY options**, built on the **Zerodha Kite Connect** API.

Code repository: [VictorG1422/trading_platform](https://github.com/VictorG1422/trading_platform)

!!! note "Current status"
    The application framework is complete, but the actual entry/exit
    trading strategy is not. That's intentional -- see
    [Safety Mechanisms & Roadmap](safety-and-limitations.md#what-remains-to-be-implemented)
    for exactly what's left and why. Everything ships safe-by-default:
    paper trading, with live order execution gated off until a strategy
    exists and three separate environment flags are set.

## New here? Read this first

[**Flow & Worked Examples**](flow-and-examples.md) explains, with a
diagram and real-number walkthroughs, exactly what happens from the
moment a price tick arrives to the moment a trade is opened, rejected, or
double-checked against the broker. Read that page before anything else
below -- it will make every other page easier to follow.

## Documentation map

| Page | What it covers |
| --- | --- |
| [Flow & Worked Examples](flow-and-examples.md) | The end-to-end pipeline, explained once with a diagram, then walked through with four concrete scenarios. |
| [Architecture & Data Flow](architecture.md) | Design decisions, instrument/underlying support, and repository layout. |
| [Module Reference](module-reference.md) | Every source file and what it's responsible for. |
| [Setup & Configuration](setup.md) | Installation, every environment variable, and the AWS Secrets Manager integration. |
| [Operations](operations.md) | Running in paper mode, Docker, PostgreSQL/Redis setup, migrations, scripts, logging. |
| [Testing](testing.md) | How to run the unit and integration test suites. |
| [Safety Mechanisms & Roadmap](safety-and-limitations.md) | Built-in safety guarantees, and what's left to build. |
| [Strategy Specification](strategy-specification.md) | The fill-in-the-blanks template a real strategy must be specified against. |

## Phase build log

The project was built in 11 numbered phases, each adding one layer on
top of the last. For each phase, two documents are kept:

- **What was asked for** -- the original prompt, saved verbatim under
  [Phase Specs](specs/00-project-initialization.md).
- **What was actually built** -- a detailed write-up of the result, under
  [Phase Writeups](phases/phase-01-foundation.md).

| Phase | Theme |
| --- | --- |
| 1 | Configuration, logging & application foundation |
| 2 | PostgreSQL database layer |
| 3 | Redis state & cache layer |
| 4 | Broker integration & market data foundation |
| 5 | Instrument discovery & option chain foundation |
| 6 | Market data & technical analysis engine |
| 7 | Trading strategy & signal engine |
| 8 | Risk & position management |
| 9 | Order execution |
| 10 | Broker reconciliation & state consistency |
| 11 | Backtesting & paper trading engine |
