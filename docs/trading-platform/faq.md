# FAQ

Short, direct answers. Each links to the page with the full explanation.

**Why didn't my strategy take a trade today?**
Almost always one of: the regime/macro gate didn't agree with a direction
(no trend, or `v2_setup` regime was CHOP/NEUTRAL), no trigger fired
(`v2_setup`) or the score never crossed the threshold (`v1_score`), a
filter blocked it (`v2_setup`), or it fired and was then rejected by the
risk engine (check the signal's risk decision — every rejection has a
`reason`). See [Flow, Strategy Logic & Diagrams](flow-and-examples.md#3-how-v1_score-decides-to-trade-the-original-strategy)
for both strategies' logic, and the dashboard's Risk Decisions page for
the exact reason a specific signal was blocked.

**What's the difference between `v1_score` and `v2_setup`, and which
should I use?**
`v1_score` is the original weighted pattern-confluence scorer — many
indicators/patterns vote, weighted, and a trade fires once the total score
crosses a threshold. `v2_setup` has no score at all — it's a regime state
machine (TREND/TREND_AGING/CHOP/NEUTRAL) gating three independent entry
triggers and five independent filters, each separately toggleable. Neither
is "the correct" one; back-test both against your own instruments and
compare. See [section 3-4 of Flow, Strategy Logic & Diagrams](flow-and-examples.md#3-how-v1_score-decides-to-trade-the-original-strategy)
for the full logic of each. Switch between them with the `strategy_mode`
setting (**Regime, Triggers & Filters (v2)** group on the settings page).

**What happens if I stop a backtest halfway through?**
It isn't cancelled or discarded. The engine finishes the candle it's on,
force-closes any open simulated positions at that candle's close
(`exit_reason=BACKTEST_STOPPED`), and saves a `STOPPED` result — fully
viewable and exportable, exactly like a completed run. See
[Flow, Strategy Logic & Diagrams, section 6](flow-and-examples.md#6-worked-example-running-a-backtest-from-the-dashboard).

**Why does the app say `WAITING_FOR_MARKET`?**
That's normal, not an error — it means the process started (or is
sitting) outside NSE trading hours (weekend, holiday, or before 09:15/
after 15:30 IST) and is waiting for the next session instead of exiting.
It auto-resumes at the next open with no restart needed. Check
`GET /api/status` for the exact reason and the next-open time. See
[Flow, Strategy Logic & Diagrams, section 1](flow-and-examples.md#1-the-process-never-exits-because-the-market-is-closed).

**How do I add a new instrument to the watchlist?**
Sync instruments (`python scripts/download_instruments.py`), then on the
dashboard's **Instruments & watchlists** page search for the synchronized
row and add it to the options or equity watchlist, then save with the
admin token. See [How-To: before you start](how-to.md#before-you-start-two-one-time-steps).

**What does the daily circuit breaker actually block?**
Once a configurable number of consecutive *closed, losing* trades happens
in one trading day (default 2), new entries are either blocked for the
rest of the day or sized at half quantity, depending on the configured
action. A winning trade resets the streak; the streak also resets every
new trading day. Open (not-yet-closed) positions never count toward it.
See [Flow, Strategy Logic & Diagrams: the daily circuit breaker](flow-and-examples.md#the-daily-circuit-breaker).

**Can I run two backtests at once?**
No — only one backtest runs at a time per process; a second request while
one is active/queued gets a clear 409 response instead of silently
queueing or corrupting the first run.

**Where do I see the risk checks that rejected a signal?**
The dashboard's **Risk Decisions** page, and the per-run **Signals & Risk
Audit** export sheet for a backtest — every signal's full ordered checklist
is recorded with a pass/fail/skip status and a plain-language detail for
each check, not just the final reason.

**Why does every number look rounded to 2 decimal places?**
That's deliberate and purely cosmetic (M5's display rule) — logs, the API,
the dashboard, and exports all round at the point of output. None of the
actual indicator/scoring/risk math is ever rounded internally, so this
never changes what a signal would have done.

**I set `entry_mode` to `break_of_signal_bar_high` or `pullback_entry` but
live/paper trading still fills at the next bar's open — is that a bug?**
No — as of this writing, those two conditional entry modes are implemented
in the **backtest engine only**; live and paper trading always use
`next_open` regardless of the setting. See
[Safety Mechanisms & Roadmap](safety-and-limitations.md) for the full list
of current gaps like this one.

**I changed a setting but nothing happened.**
Most settings are only read once, at process startup — stop the running
process and start it again. See [How-To: changing settings](how-to.md#how-to-change-settings).

**My backtest/paper session shows a profit or loss, but "total trades" is
zero — is that a bug?**
No. The P&L reflects an open position being marked to the latest price;
it only becomes a counted "trade" once it's closed.

**Can I trade NIFTY/BANKNIFTY itself?**
Not directly — only its futures, its options, or an ETF that tracks it.
Select the index's EQ/index reference row in the options watchlist and
let the system trade the corresponding CE/PE.
