# History — how this was built

This project was built in two spec-driven rounds. Each round's original
prompt is kept verbatim here as the historical record of *what was asked
for*; the rest of the docs (outside this `history/` folder) describe
*what the system does today*, kept current as the code changes.

- **Phases 1-11** (original build): configuration/logging foundations,
  PostgreSQL, Redis, Kite broker + market data, instrument/option-chain
  discovery, the technical-analysis engine, the strategy/signal engine,
  risk and position management, order execution, broker reconciliation,
  and backtesting/paper trading. See [Phase Specs](specs/00-project-initialization.md).
  The narrative "what was actually built" write-ups that used to sit
  alongside these specs were removed from this repository before this
  cleanup pass (only the specs survived); nothing in the live docs links
  to them anymore.
- **M0-M15 upgrade** (most recent round): restructured the app down to two
  startup modes (paper/live — no more "backtest" as a startup mode), added
  a market-hours state machine so the process never exits just because
  NSE is closed, rebuilt the Backtest tab around an on-demand background
  job with a calendar, Run/Stop, live progress, and a 10-sheet XLSX/CSV
  export, enforced a single 2-decimal output rule everywhere, added a
  daily circuit breaker, added a second strategy engine (`v2_setup`:
  a regime state machine plus independent entry triggers and filters,
  alongside the original `v1_score` weighted-pattern-confluence scorer),
  and added idea-level risk budgets, configurable entry modes, and
  partial-exit/time-stop trade management. The rest of this documentation
  set describes the result; no separate spec document for this round is
  kept here, since the current docs already describe the outcome in full.

**A note on illustrative numbers**: a few places in these docs (the
circuit-breaker explanation, in particular) reference example trade
counts or P&L figures from the project owner's own manual trading
history, used purely to illustrate *why* a feature exists. These are
never claimed as verified output of this codebase's test suite —
where that distinction matters, the text says so explicitly.
