---
name: document-strategy
description: Write structured strategy documentation - hypothesis, universe, entry/exit rules, sizing, backtest results, costs, risks. Use when writing up a trading strategy spec for review or recording/updating a strategy after a parameter change.
---

# Strategy Documentation Template

Sections to always include, in order:
1. **Hypothesis** - the market inefficiency exploited and why it should persist.
2. **Universe** - exchange, instrument type, filter criteria (liquidity, F&O eligibility, index membership).
3. **Signals & entry rules** - table of signal/condition/weight, composite score threshold.
4. **Position sizing** - risk per trade (% of NAV), stop placement, max concurrent positions, product type (MIS/CNC/NRML).
5. **Exit rules** - target, hard stop, time stop (e.g. 15:15 IST for MIS), trailing logic.
6. **Parameters** - table of tunable values with notes.
7. **Backtest results** - CAGR, Sharpe, max DD, win rate, trade count, by period.
8. **Costs assumed** - brokerage, STT, slippage, total round-trip estimate.
9. **Risks/edge cases** - circuit halts, expiry-day liquidity, high-impact news, auto-square-off miss.
10. **Change log** - date/change/reason.

Keep each section terse (tables over prose); fill with real numbers from the actual backtest run, not placeholders, before publishing.
