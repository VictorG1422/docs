# Trading Strategy Specification — PROPOSAL / TEMPLATE (NOT YET DEFINED)

> **Status: AWAITING INPUT.** This document is a structured template, not an
> implemented strategy. No section below has been filled in with real
> trading rules. Nothing in `src/trading_system/strategy/` has been changed
> based on this file — `NoOpStrategy` remains the active strategy
> implementation until every section here is completed and reviewed.
>
> This file exists because Phase 7A ("Actual Trading Strategy Specification
> and Implementation") requires a complete, explicit strategy specification
> before any entry/exit logic can be implemented. A repository-wide search
> (`queries/`, source, tests, README, git history) found no strategy rules,
> indicator thresholds, entry/exit conditions, or option-selection rules
> anywhere — only the explicit instruction *"Do not invent a strategy"*
> (see [the project-initialization spec](specs/00-project-initialization.md)).
> Per the Phase 7A instructions, arbitrary rules must not be invented;
> instead this template is provided for the project owner to complete.

## How to use this document

Fill in every section below with concrete, unambiguous, testable values.
Avoid vague language ("strong trend", "good momentum", "bullish") —
every condition must be expressible as a concrete comparison the code can
evaluate (e.g. `rsi_14 > 60`, `ema_9 crosses_above ema_21`,
`close > previous_candle.high`). Once complete, this document becomes the
authoritative input for implementing a real `BaseStrategy` subclass.

---

## A. Instruments

Which underlying instrument(s) does this strategy trade?

- [ ] NIFTY
- [ ] BANK NIFTY
- [ ] Other: `______`

*(Do not assume both if only one is intended. List every instrument the strategy actually trades.)*

## B. Trading session

- Allowed trading start time: `______` (IST)
- Allowed trading end time: `______` (IST)
- Are signals allowed near market close (e.g. last N minutes before 15:30 IST)? `______`
  - If not, define the cutoff: `______`
- Does the strategy evaluate throughout the session, or only during specific windows? `______`
  - If specific windows, list them: `______`

## C. Primary timeframe

- Candle interval: `______` minutes (e.g. 1 / 5 / 15)
- Justification / source of this choice: `______`

## D. Confirmation timeframe

- Is a second (confirmation) timeframe required? `Yes / No`
- If yes, candle interval: `______` minutes, and what it confirms: `______`
- If no: this strategy is explicitly **single-timeframe**.

## E. Indicators

List **every** indicator the strategy uses. Add rows as needed — do not
leave placeholders unfilled if the indicator is actually used, and do not
list an indicator that isn't used.

| Indicator | Period | Source price | Timeframe | Required historical observations |
|---|---|---|---|---|
| `______` | `______` | `______` (e.g. close) | `______` | `______` |

*(Every indicator listed here must already exist in `market_data/indicators.py` (SMA, EMA, RSI, MACD, ATR, VWAP, Bollinger Bands) or be explicitly identified as a new indicator to add to that module — never recalculated inside the strategy.)*

## F. Entry conditions

List each condition individually, then define how they combine.

1. Condition 1: `______`
2. Condition 2: `______`
3. Condition 3: `______`

Combination logic (`AND` / `OR` / conditional — describe exactly): `______`

*(Every condition must be a concrete, measurable comparison — not a description.)*

## G. Confirmation

What exactly confirms an entry once the conditions in section F are met?

- [ ] Candle close (which candle, which condition on it: `______`)
- [ ] Indicator crossover (which indicators, which direction: `______`)
- [ ] Volume condition: `______`
- [ ] Price relationship: `______`
- [ ] Multi-timeframe confirmation (only if section D is "Yes"): `______`
- [ ] Other: `______`

## H. CE/PE selection (only if this strategy trades options)

- How is CALL vs PUT selected? `______`
- How is expiry selected (e.g. nearest weekly, nearest monthly)? `______`
- How is strike selected? `______`
  - ATM? `Yes / No`
  - ITM/OTM? `______`
  - Number of strikes away from ATM: `______`
  - How is "strike distance" measured (points, % of spot, strike-step multiples)? `______`
- Confirm: contract selection must go through Phase 5 `OptionChainService` — never a hardcoded token/tradingsymbol.

*(If this strategy does not trade options, state so explicitly and skip this section.)*

## I. Signal conditions

- Exact condition(s) under which the engine returns `BUY`: `______`
- Exact condition(s) under which the engine returns `SELL`: `______`
- Exact condition(s) under which the engine returns `NO_SIGNAL` (i.e. `BaseStrategy.evaluate()` returns `None`): `______`
- Is `HOLD` used? `Yes / No` — if yes, its exact meaning (distinct from `NO_SIGNAL`): `______`

*(Reminder: a signal is a request, never an executed order. Phase 8/9 decide whether/how it becomes a trade.)*

## J. Signal invalidation

Under what conditions does an otherwise-valid setup become invalid before a signal is generated (e.g. price moves past a level, a confirmation window expires, market data goes stale)? `______`

## K. Duplicate signals

- Can the same setup/candle generate more than one signal? `Yes / No`
- If "No" (the default via the existing Phase 3 `DeduplicationStore`): confirm no change needed.
- If "Yes": define exactly under what circumstances a repeat signal for the same candle is allowed: `______`

## L. Cooldown

- Is a cooldown period required between signals? `Yes / No`
- If yes, exact duration and scope (per-instrument? per-strategy?): `______` seconds
- Source/justification for this duration (must not be invented/optimized in this phase): `______`

---

## Sign-off

- Specified by: `______`
- Date: `______`
- Reviewed against `queries/00_project_initialization.md` and `queries/07a_phase7a.md`: `Yes / No`

Once every section above is completed, re-run Phase 7A (or request the
implementation directly) to replace `NoOpStrategy` with the specified
rules using the existing Phase 7 framework (`MarketContext`,
`StrategyEngine`, `SignalValidator`, `StrategyStateStore`,
`DeduplicationStore`, `OptionChainService`) — no new framework code
should be required, only a new `BaseStrategy` subclass and configuration.
