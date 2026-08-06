# DI Proof of Performance

**An autonomous AI trading system. This repository is the public proof.**

DI has been running live since **2026-08-06**. Every day, this repository receives a new performance report — automatically generated, automatically pushed, never edited.

## What you'll find here

| Directory | What's inside |
|-----------|--------------|
| [`daily/`](daily/) | One report per day — equity, positions, trades, agent health |
| [`trades/`](trades/) | Settled trade records with entry/exit reasons |
| [`metrics/`](metrics/) | Rolling PF, Sharpe, win rate |
| [`incidents/`](incidents/) | Every system incident — what broke, when, how it was fixed |

## Why this exists

Most crypto trading bots show you a backtest. DI shows you **what actually happened every single day since it started.**

- ✅ Live trading only — no backtest-only results
- ✅ Every trade documented with entry reason, exit reason, and version
- ✅ Every incident recorded with timestamp and fix commit
- ✅ Git history proves nothing was edited after the fact

## How it works

```
DI scans markets every minute
  → evaluates RSI + sentiment + regime
  → executes trades on OKX with automated risk controls
  → logs everything
  → pushes this report at 23:55 CST daily
```

## Current status

See the latest report: [`daily/`](daily/)

**Strategy:** hrscan v6.1.1 — Multi-timeframe RSI momentum  
**Exchange:** OKX perpetual futures  
**Risk:** 5% per trade, auto SL/TP, Kill Switch, circuit breaker  
**Started:** 2026-08-06 · $517 initial capital

---

*No promises. No backtests. Just records.*
