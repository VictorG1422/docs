---
name: trade-journal
description: Write a live trading session journal - market context, executed trades, execution quality vs backtest expectations, anomalies, session P&L. Use when documenting a live trading session or comparing expected vs actual fills.
---

# Trade Journal Structure

1. **Market context** - NIFTY 50 open/close/change, India VIX trend, broad market regime (bullish/bearish/choppy/trending).
2. **Trades executed table** - symbol, side, qty, signal time, entry time, entry/exit price, slippage (bps), PnL, notes.
3. **Execution quality** - expected (backtest) vs actual: avg slippage bps, fill rate %, avg entry lag (sec), whether auto-square-off triggered.
4. **Anomalies** - order rejections (with reason), API/token exceptions, missed signals (fired but not executed), unusual conditions (circuit halt, thin liquidity).
5. **Session P&L** - gross/costs/net, split realised vs unrealised EOD.
6. **Action items** - parameters to review, code fixes needed, follow-ups.

Compute slippage as `(actual_fill - signal_price) / signal_price * 10000` bps and compare directly against the assumption in `costs.py`; flag execution lag > 2s as an anomaly worth investigating.
