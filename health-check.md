# DI Strategy Health Check — Free

**I built a transparent AI risk layer for crypto trading bots.**
Before you lose real money, let me find what your backtest missed.

---

## What I Check

- **Regime Detection** — Does your strategy only work in trending markets?
- **Hidden Exposure** — Are you accidentally over-concentrated?
- **Failure Scenarios** — What breaks your bot in live trading?
- **Execution Risks** — Slippage, API failures, duplicate orders

---

## How It Works

1. You share your strategy (Freqtrade config, TradingView script, Python logic, or just describe it)
2. I run it through the same framework I use for my own live trading system
3. You get a **free risk report** — no strings, no payment

---

## Submit Your Strategy

Open a GitHub Issue here with:
- Strategy description (entry/exit/size rules)
- Market and timeframe
- Any backtest results you have

→ [Create Issue](../../issues/new?title=Health+Check:+[strategy+name]&body=**Strategy:**%0A**Market:**%0A**Timeframe:**%0A**Backtest results (if any):**%0A**Biggest concern:**)

I'll reply within 24h with a full risk diagnosis.

---

## Example Report

> **Strategy:** EMA 21/55 cross + RSI 30/70
> **Finding 1:** Works only in trending regime — fails 80% of the time in choppy markets
> **Finding 2:** No circuit breaker — one API failure = infinite duplicate orders
> **Finding 3:** Hidden drawdown risk from correlated positions
> **Recommendation:** Add regime filter + Kill Switch. Reduce exposure 70% in current CHOPPY market.

---

## Who Am I?

I run **hrscan v6** — an autonomous AI trading agent on OKX with real money.
Every trade, incident, and fix is publicly recorded here in this repo.

- [Daily Reports](daily/)
- [System Incidents](incidents/)
- [OKX Copy Trading #9D58C06](https://www.okx.com/copy-trading)

---

## After the Free Check

If you want ongoing monitoring:
- **$49/mo** — Daily regime + risk alerts for your strategy
- **$149** — Full autonomous deployment with Kill Switch

But first — let's find out if your strategy survives real markets.
