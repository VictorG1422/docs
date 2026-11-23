# Algo Trading System

A production-oriented, modular-monolith algorithmic trading system for
**NSE NIFTY / BANK NIFTY options**, built on the **Zerodha Kite Connect**
API, with a full backtesting engine and dashboard built in.

Code repository: [VictorG1422/trading_platform](https://github.com/VictorG1422/trading_platform)

!!! note "Current status"
    The application runs one configured strategy end to end — a
    regime-state-machine-and-trigger system (regime detection, independent
    entry triggers, and entry filters) — plus a `no_op` strategy that
    deliberately disables signal generation. It is implemented and
    covered by the automated test suite. That is **not** the same as
    "approved for live trading with real money": see
    [Safety Mechanisms & Roadmap](safety-and-limitations.md) for exactly
    what independent validation is still expected before you trust it
    with capital you can't afford to lose. Everything ships
    safe-by-default: paper trading, with live order execution gated off
    until you explicitly flip three separate switches (see
    [How-To: Going live](how-to.md#how-to-go-live-placing-real-orders--read-this-first)).

## New here? Read this first

[**Flow, Strategy Logic & Diagrams**](flow-and-examples.md) explains, in
plain language with diagrams, what happens from the moment the app starts
to the moment a trade is opened, rejected, or double-checked against the
broker — including how the strategy actually decides to buy or sell, and
how the risk engine's ordered checklist works. Read that page before
anything else below.

**Want to actually run something?** Skip straight to the
[**How-To Guide**](how-to.md) — step-by-step instructions for running a
backtest from the dashboard (calendar, Run/Stop, progress, export), running
paper trading against live data, changing settings, and troubleshooting.

**Have a specific question?** Check the [**FAQ**](faq.md) first — it's the
fastest way to answer things like "why didn't my strategy trade today?" or
"what does WAITING_FOR_MARKET mean?".

## Documentation map

| Page | What it covers |
| --- | --- |
| [Flow, Strategy Logic & Diagrams](flow-and-examples.md) | The end-to-end pipeline with diagrams, plain-language strategy/risk logic, and worked examples. |
| [How-To Guide](how-to.md) | Practical, step-by-step instructions: running a backtest, running paper trading, changing settings, troubleshooting. |
| [FAQ](faq.md) | Short, direct answers to the questions operators actually ask. |
| [Architecture & Data Flow](architecture.md) | Design decisions, instrument/underlying support, and repository layout. |
| [Module Reference](module-reference.md) | Every source file and what it's responsible for. |
| [Setup & Configuration](setup.md) | Installation, every environment variable, and the AWS Secrets Manager integration. |
| [Operations](operations.md) | Running in paper mode, Docker, PostgreSQL/Redis setup, migrations, scripts, logging. |
| [Testing](testing.md) | How to run the unit and integration test suites. |
| [Safety Mechanisms & Roadmap](safety-and-limitations.md) | Built-in safety guarantees, and what's left to build. |
| [Strategy Specification](strategy-specification.md) | The fill-in-the-blanks template for specifying a brand-new, custom strategy (separate from the built-in one). |
