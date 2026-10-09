# Phase 7 — Trading Strategy & Signal Engine

Phase 7 builds the strategy/signal-generation framework on top of the
Phase 6 market-data/indicator engine. **It only ever terminates at
"signal generated" -- it never places an order, never manages risk or
position sizing, and never executes a trade.** Phase 8+ owns everything
after a signal is persisted. Phase 7A (a follow-up pass over the same
phase) was run specifically to replace the placeholder strategy with a
real one -- both are documented together here as a single phase, since
7A changed nothing about the architecture below, only confirmed its
current status.

**No trading-strategy specification exists anywhere in this project** --
the original project-initialization prompt explicitly says *"Keep the
actual entry/exit rules as TODOs... Do not invent a strategy."* Phase 7
was re-checked for this specifically in the Phase 7A pass: the entire
repository (source, tests, comments, README, git history) was
re-searched for entry/exit rules, indicator thresholds, CE/PE-selection
rules, or any other strategy documentation, and none exist. Per Phase
7A's explicit instructions, no strategy logic was invented as a result.

Phase 7 therefore ships the complete framework wired end-to-end, with
`NoOpStrategy` (a placeholder that never returns a decision) standing in
for a real strategy. In its place, [the strategy specification](../strategy-specification.md)
was added: a structured, fill-in-the-blanks template covering every
element a real strategy must define (instruments, trading session,
primary/confirmation timeframe, indicators with period/source/timeframe,
entry conditions with explicit AND/OR logic, confirmation, CE/PE
selection, signal conditions, invalidation, duplicate-signal behavior,
cooldown). `NoOpStrategy` remains the active strategy implementation
until that document is completed -- no entry/exit rule, indicator
threshold, or option-selection rule was invented in its place.

**Data flow:**

```
CandleAggregator (Phase 6, candle-completion event -- never per-tick, never a timer)
        │
        ▼
StrategyEngine.evaluate_candle()
        │  MarketState.status() freshness gate -- never evaluates on stale/unavailable data
        │  DistributedLock (per strategy+instrument+timeframe+candle-timestamp)
        ▼
MarketContext  (candles up to & including this candle -- never a future one)
        │
        ▼
BaseStrategy.evaluate(context) -> StrategyDecision | None
        │
        ▼
dedup check (DeduplicationStore) -> cooldown check (StrategyStateStore) -> SignalValidator
        │
        ▼
Signal --> SignalRepository (PostgreSQL, status="RISK_CHECK_PENDING")
        --> StrategyStateStore (Redis: last_signal_type/timestamp, trade_count)
```

- `trading_system/strategy/context.py::MarketContext` -- an immutable,
  per-evaluation snapshot: `underlying`/`instrument_token`/`tradingsymbol`/
  `timeframe_minutes`/`timestamp` (the just-completed candle's own
  timestamp -- never "now"), `current_price`, `candles` (completed
  candles up to and including `timestamp`), `market_status`, and
  `strategy_state` (previously saved `StrategyStateStore` state). Also
  `confirmation_candles` (Phase 7's multi-timeframe extension): completed
  candles for each of `StrategyConfig.confirmation_timeframes_minutes`,
  keyed by timeframe minutes -- empty unless a strategy configured extra
  timeframes. A strategy never receives a Redis/PostgreSQL/Kite handle directly.
- `trading_system/strategy/base.py::BaseStrategy` -- the Phase 7
  interface: `evaluate(context: MarketContext) -> StrategyDecision | None`.
  A pure function of `context`; "no signal" is expressed by returning
  `None`, never a placeholder decision. The original tick-driven
  `Strategy` ABC (Phase 1 scaffold, never wired anywhere) is untouched.
- `trading_system/strategy/decision.py::StrategyDecision` -- a
  strategy's raw output (`direction`, `reason`, `price`,
  `indicator_snapshot`, `metadata`, optional `option_contract`) *before*
  it becomes a formal `Signal` -- a strategy never builds its own signal
  ID, dedup key, or persists anything itself.
- `trading_system/strategy/no_op_strategy.py::NoOpStrategy` -- the Phase 7
  placeholder; `evaluate()` always returns `None`. Kept as the default
  strategy implementation; `TechnicalScoringStrategy` (below) is an
  available, generic alternative -- neither is wired into `Application`
  automatically (see below).
- **`TechnicalScoringStrategy` (multi-timeframe pattern/indicator
  confluence scoring)** -- a production-standard, generic technical-
  analysis strategy, added on explicit request as an alternative to
  `NoOpStrategy`. Not this project's specific (still-undefined) trading
  rules -- see [the strategy specification](../strategy-specification.md) -- but a reusable,
  fully-configurable scoring engine:
  - `trading_system/strategy/patterns.py` -- pure, stateless pattern/
    indicator detectors over completed candles (mirrors
    `indicators.py`'s conventions: `Decimal` throughout, `None` on
    insufficient data, never raises). Candlestick patterns: bullish/
    bearish engulfing, hammer, shooting star, doji, morning/evening
    star, three white soldiers/three black crows. Swing-point chart
    patterns: higher-highs/higher-lows (or lower-highs/lower-lows) trend
    structure, and **head-and-shoulders / inverse head-and-shoulders**
    (`_swing_highs`/`_swing_lows` find local fractal swing points --
    a candle whose high/low is the most extreme within a configurable
    window on both sides; three consecutive swings with a higher/lower
    middle "head" and two roughly-level "shoulders" -- within
    `shoulder_tolerance`, default 5% -- form the pattern, but it is only
    reported once price actually closes through the neckline, the
    higher/lower of the two troughs/peaks flanking the head -- an
    unconfirmed shape is never itself a signal). **F&O-specific**
    (`oi_buildup`, `volume_spike`) -- added specifically for NIFTY/
    BANKNIFTY F&O: `oi_buildup` classifies the latest candle's price
    change against its open-interest change into the standard NSE
    convention (long buildup / short buildup / short covering / long
    unwinding -- requires `Candle.open_interest`, `None` when absent,
    never fabricated); `volume_spike` confirms a candle's direction is
    backed by volume well above its recent average (real participation,
    not noise) -- supporting confirmation only, never a standalone
    directional signal. Indicator-derived (supporting confirmation): RSI
    overbought/oversold, MACD signal-line crossover, EMA(9/21)
    golden/death cross, Bollinger Band mean-reversion touch, VWAP bias.
    Each detector returns a `PatternMatch` (`name`, `direction`,
    `strength` 0-100) -- detection confidence only, never an importance
    weight.
  - **Patterns are weighted above pure indicators by default**
    (`scoring._DEFAULT_WEIGHTS`) -- e.g. `head_and_shoulders` (32),
    `star_reversal`/`three_candle_continuation` (28-30), `engulfing`
    (25), `oi_buildup` (24) all outweigh `macd_crossover` (14),
    `rsi_extreme`/`ema_trend_cross` (12), `bollinger_reversion` (10),
    `volume_spike` (8), `vwap_bias` (6).
    Indicators remain in the confluence as supporting confirmation, at a
    reduced weight, per explicit product preference: a pattern is a
    direct read of price structure, while an indicator is a lagging
    derived result. Fully overridable via `ScoringConfig.weights`.
  - `trading_system/strategy/scoring.py::TechnicalScorer`/`ScoringConfig`
    -- combines every pattern match from the primary timeframe **and**
    every configured confirmation timeframe ("patterns should check in
    multiple timeframes") into one `-100..100` score using per-pattern
    weights (`ScoringConfig.weights`, generic/conventional confluence-
    style defaults, fully overridable); confirmation timeframes
    contribute at a lower weight multiplier (`confirmation_weight_multiplier`,
    default 0.6x). A signal only fires once the score crosses
    `buy_score_threshold`/`sell_score_threshold` (default 50) --
    confluence of multiple independent signals, never a single
    indicator alone (unless its weight alone exceeds the threshold). A
    separate EMA-based trend filter on each confirmation timeframe can
    **veto** a signal outright when it conflicts with the higher-
    timeframe trend (`require_confirmation_alignment`, default `True`)
    -- the widely-cited "trade with the higher-timeframe trend" rule,
    independent of the score itself.
  - `trading_system/strategy/technical_scoring_strategy.py::TechnicalScoringStrategy`
    -- the `BaseStrategy` implementation wrapping `TechnicalScorer`;
    returns `None` below threshold/when vetoed, otherwise a
    `StrategyDecision` whose `indicator_snapshot` records the total
    score, every matched pattern (name/direction/strength/timeframe),
    and whether a confirmation-timeframe veto applied -- fully
    auditable. Slots into the existing framework unchanged:
    `StrategyEngine` still owns signal identity/dedup/cooldown/
    validation/persistence.
  - Not wired into `Application`/`NoOpStrategy`'s place automatically --
    same deliberate "framework exists, nothing forces it live" choice
    as the rest of Phase 7.
- `trading_system/strategy/config.py::StrategyConfig` -- named,
  documented, testable configuration: `strategy_name`, `underlying`,
  `instrument_token`, `timeframe_minutes`, `cooldown_seconds` (0 =
  disabled -- no invented "profitable" default), `candle_lookback`,
  `persist_signals`, `lock_ttl_seconds`, `confirmation_timeframes_minutes`
  (additional timeframes -- e.g. `(15, 60)` alongside a 5-minute primary
  -- whose completed candles are also attached to each `MarketContext`;
  empty by default, an explicit per-strategy opt-in), and an opaque
  `extra` dict for a concrete strategy's own parameters (indicator
  periods/thresholds) once a real specification justifies naming them.
- `trading_system/strategy/strategy_engine.py::StrategyEngine` -- the
  orchestrator. Registers itself as a `MarketDataService` candle-
  completion listener (`add_candle_completion_listener`, Phase 6,
  extended here) -- the *only* evaluation trigger; there is no per-tick
  or timer-based evaluation. On each completed candle it: checks
  `MarketState`/`MarketDataStatus` freshness (never evaluates on
  `STALE`/`UNAVAILABLE` data); acquires a `DistributedLock` scoped to
  `strategy+instrument+timeframe+candle timestamp` (bounded, released
  immediately after); builds the `MarketContext`; calls
  `strategy.evaluate()` inside a try/except so a strategy exception (or
  any internal failure) is logged (`STRATEGY_ERROR`) and produces *no*
  signal, never a false BUY/SELL; computes the deterministic dedup key
  and checks it via the Phase 3 `DeduplicationStore`; checks cooldown via
  `StrategyStateStore`; validates the candidate `Signal`
  (`signal_validator.py`); and, if accepted, persists it and updates
  strategy state.
- `trading_system/strategy/signal_factory.py` -- `build_signal_dedup_key()`
  (`strategy:instrument:timeframe:candle_timestamp` -- deterministic,
  re-evaluating the same candle always yields the same key) and
  `build_signal()` (turns a `MarketContext` + `StrategyDecision` into a
  formal `Signal`). A strategy never invents its own identity.
  **`Signal.instrument_token`/`tradingsymbol`/`expiry`/`strike`/`option_type`
  come from `StrategyDecision.option_contract` when set, never from the
  underlying's own instrument/context** -- see "Underlying signal ->
  tradable option contract" below for why this matters.
- `trading_system/strategy/signal_validator.py::validate_signal()` -- a
  pure function (no I/O) returning a list of failure reasons (empty =
  valid): missing/expired instrument, non-fresh market data, missing
  trading symbol, non-positive price, a timestamp too far in the future,
  or a missing dedup key. A validation failure is an everyday outcome
  (most candles produce no valid signal), never an exception.

**Underlying signal -> tradable option contract.** In NSE F&O, NIFTY/
BANKNIFTY itself is never bought/sold -- only its CE/PE option contracts
are. A strategy therefore *analyzes* the index/futures chart
(`StrategyConfig.instrument_token`/`underlying`) but the actual tradable
instrument the resulting `Signal` must reference is a specific option
contract. This translation is **engine-level infrastructure, not a
strategy concern** (keeps `BaseStrategy.evaluate()` a pure function of
`MarketContext`, with no direct database/option-chain access):

- `trading_system/strategy/option_selector.py::OptionSelector` -- given a
  direction (`BUY`/`SELL`) and the current spot price, selects a concrete
  `OptionContract` via the existing Phase 5 `OptionChainService`/
  `ExpiryService` (never a hardcoded/guessed token). Since this project
  only ever **buys** options (never writes/sells naked options): a
  bullish (`BUY`) view buys a **call (CE)**; a bearish (`SELL`) view buys
  a **put (PE)** -- there is no "sell a call to go bearish" path.
  `OptionSelectionConfig` controls strike selection (`ATM` by default --
  the most liquid, balanced choice for buying options; `OTM`/`ITM` via a
  configurable strike-step *offset*, applied as list-index steps into the
  actual available strikes for that expiry, never a hardcoded price
  step -- correct regardless of an underlying's actual strike spacing,
  e.g. NIFTY 50-point vs BANKNIFTY 100-point) and expiry selection
  (nearest by default; `min_days_to_expiry` can require skipping very
  near-dated/0DTE expiries -- a common practice to avoid extreme
  same-day gamma/pin risk -- opt-in, `0` i.e. disabled by default).
- `trading_system/strategy/strategy_engine.py::StrategyEngine._resolve_option_contract()`
  -- called right after a strategy returns a `BUY`/`SELL` decision with
  no `option_contract` of its own: resolves the contract via
  `OptionSelector`, then fetches **that option's own live price**
  (`MarketDataService.get_current_price(contract.instrument_token)` --
  never the index's price) to become `StrategyDecision.price`. If
  contract selection fails (e.g. no expiry/strikes synchronized) or the
  option's price is unavailable (e.g. its instrument token was never
  subscribed to the tick stream), the candle produces **no signal** --
  never a fabricated price or a signal against the wrong instrument. A
  strategy that already selected its own contract, or an `OptionSelector`
  simply not configured (e.g. a strategy trading the underlying/futures
  instrument directly), passes through unchanged. The original
  underlying price used to generate the signal is preserved in
  `StrategyDecision.metadata["underlying_price"]` for audit -- it is easy
  to otherwise lose track of "the chart price that triggered this" once
  `price` becomes the option's premium.

**Signal model (extended, not duplicated):** `models/signal.py::Signal`
(the same model used since Phase 1) gained optional/defaulted Phase 7
fields -- `underlying`, `expiry`, `strike`, `option_type`,
`timeframe_minutes`, `indicator_snapshot`, `dedup_key` -- so there is a
single `Signal` type for both the original placeholder pipeline and the
new `StrategyEngine`, never a parallel model.

**Signal lifecycle:** `GENERATED` (a `StrategyDecision` was returned) ->
(duplicate/cooldown/invalid -> rejected, **never persisted** -- "do not
store unnecessary high-frequency duplicate signals") -> `VALIDATED` ->
`ACCEPTED` -> `RISK_CHECK_PENDING` (the only status Phase 7 ever writes
to PostgreSQL). Phase 8 reads `RISK_CHECK_PENDING` signals and owns every
state after that -- Phase 7 never calls risk/order/broker code.

**Duplicate-signal protection:** the deterministic dedup key
(`build_signal_dedup_key`) is checked via the existing Phase 3
`DeduplicationStore.check_and_mark()` (atomic `SET NX EX`, never a
GET-then-SET race) *before* validation/persistence -- re-processing the
same completed candle twice (e.g. after a process restart) never
produces two signals.

**Strategy state (Phase 3 `StrategyStateStore`, Redis):** keyed by
`strategy_name:instrument_token:timeframe_minutes` (a single opaque
string passed as the "strategy name" -- `StrategyStateStore` itself was
not modified). Holds `last_signal_type`, `last_signal_timestamp`, and
`trade_count`, updated only after an accepted signal. Never used for
duplicate detection (that is `DeduplicationStore`'s job) -- only for
cooldown and strategy-specific runtime state.

**Concurrency:** a `DistributedLock` (Phase 3) scoped to one candle's
evaluation is acquired before calling the strategy and released
immediately after (`lock_ttl_seconds`, default 30s) -- bounded so a
crashed process can never block future evaluations forever. This guards
against two processes evaluating the exact same completed candle
concurrently; it is not a general-purpose lock and is never held across
I/O beyond one evaluation+persist cycle.

**Look-ahead protection:** a strategy is only ever invoked with candles
completed at or before its own decision timestamp -- `MarketContext` is
built fresh, per evaluation, from `MarketDataService.get_completed_candles()`
(which never includes the still-forming candle) plus the just-completed
candle itself. There is no code path by which a strategy can observe a
candle, tick, or indicator value dated after `context.timestamp`.

**Option-chain interaction:** `StrategyDecision.option_contract` is an
optional `OptionContract` (Phase 5) a concrete strategy can set if its
specification requires selecting a specific CE/PE contract; `NoOpStrategy`
never sets it. `signal_factory.build_signal()` copies the contract's
`tradingsymbol`/`expiry`/`strike`/`option_type` onto the `Signal` when
present.

**Database changes:** `migrations/versions/0005_signal_status.py` adds a
plain `status` string column (default `"RISK_CHECK_PENDING"`) to the
existing `signals` table -- a full enum type was deliberately avoided
since the lifecycle is still evolving and Phase 7 only ever writes one
value.

**Testing:** `tests/unit/test_strategy_engine.py` (no-signal/valid-signal
paths, stale-market-data gate, missing/expired instrument, duplicate
signal, cooldown, strategy exception isolation, invalid price, strategy
state update, look-ahead protection, determinism, `start()` wiring,
wrong-instrument/timeframe candle rejection, configured/unconfigured
confirmation-timeframe candle attachment, option-contract resolution --
BUY->CE/SELL->PE, the option's own live price replacing the index price,
no-signal when the option's price is unavailable or selection fails,
an already-strategy-selected contract passing through unchanged),
`tests/unit/test_no_op_strategy.py`,
`tests/unit/test_signal_factory.py` (dedup-key determinism, field
mapping, `instrument_token` sourced from the option contract), `tests/unit/test_signal_validator.py` (every rejection reason),
plus an extension to `tests/unit/test_market_data_service.py`
(`add_candle_completion_listener`). `tests/unit/test_option_selector.py`
(BUY->CE/SELL->PE, ATM/OTM/ITM strike selection in both directions,
strike-offset clamping at the available-strikes boundary, `min_days_to_expiry`
skipping a too-near expiry, propagating Phase 5 lookup errors rather than
swallowing them). `tests/unit/test_patterns.py` (every
candlestick/indicator detector, hand-derived deterministic candle/price
sequences -- including exact EMA/MACD crossover arithmetic using period=1
"identity EMA" trick for verifiable-by-hand test data, constructed
swing-point sequences for head-and-shoulders/inverse head-and-shoulders
covering confirmed-by-neckline-break, shape-present-but-not-yet-confirmed,
and insufficient-data cases, and all four OI-buildup classifications plus
volume-spike confirmation for both directions), `test_scoring.py`
(weighted confluence summation, threshold behavior, confirmation-timeframe
weight multiplier, score clamping, higher-timeframe-trend veto with a real
declining-price EMA bias), and `test_technical_scoring_strategy.py`
(`BaseStrategy` wiring, stale/missing-price guards, indicator-snapshot
construction) cover `TechnicalScoringStrategy`. `tests/integration/test_strategy_engine_integration.py`
exercises `TickProcessor -> MarketState -> MarketDataService -> StrategyEngine
-> Signal -> PostgreSQL + Redis` end-to-end against real local
Postgres/Redis (gated by `RUN_INTEGRATION_TESTS=1`, using a scripted
always-BUY test double -- never `NoOpStrategy`, since it never signals,
and never the real Kite API).

**Bug found and fixed along the way:** `CandleAggregator` (Phase 6) used
a plain `threading.Lock`, but `on_tick()` holds it for the *entire*
bucket-rollover-and-listener-invocation sequence. `StrategyEngine`'s
candle-completion listener legitimately calls back into
`get_completed_candles()` (to build `MarketContext`), which re-acquires
the same lock on the same thread -- a guaranteed self-deadlock with a
non-reentrant lock. Fixed by switching to `threading.RLock()`; Phase 6's
own listener (persist/cache) never made this callback, so the bug was
latent until Phase 7 exercised it.

See the original prompt: [specs/07-phase7.md](../specs/07-phase7.md).
