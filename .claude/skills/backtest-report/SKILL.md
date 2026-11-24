---
name: backtest-report
description: Write a stakeholder-facing backtest results report - summary metrics, drawdown analysis, trade distribution, cost impact, parameter sensitivity. Use when documenting backtest output or comparing strategy variants for review.
---

# Backtest Report Structure

1. **Summary table**: total return, CAGR, benchmark (NIFTY 50 buy&hold) CAGR, alpha, Sharpe (Rf~7%), Sortino, Calmar, max DD %, max DD duration, win rate, profit factor, avg trade R, total trades, trades/month.
2. **Equity curve** - describe shape qualitatively if no chart (e.g., steady uptrend with one major drawdown in <period>).
3. **Drawdown table** - start/end/depth/duration for each significant drawdown.
4. **Trade distribution** - long vs short counts/avg PnL, best/worst trade with dates.
5. **Cost impact table** - brokerage/STT/slippage/all-costs, each as total ₹, per-trade avg, and % of gross PnL.
6. **Parameter sensitivity** - ±20% sweep on top 2-3 parameters, Sharpe delta.
7. **Conclusions & next steps** - e.g. walk-forward validation, paper-trade period before capital allocation, regime-specific review.

Pull metrics directly from the run's stdout/JSON output; compute cost-impact % from `costs.py` totals vs gross PnL rather than estimating.
