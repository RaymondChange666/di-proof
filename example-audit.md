# Example: BTC Breakout Bot — Risk Audit

> *This is a sample report. [Get yours free →](../../issues/new?title=Health+Check:+%5Bstrategy+name%5D&body=%23%23%20Strategy%0A%0A%23%23%20Market%0A%0A%23%23%20Timeframe%0A%0A%23%23%20Biggest%20Concern%0A)*

---

**Strategy:** BTC-USDT · 4H Breakout · EMA 50/200 cross · 2% risk per trade

**Risk Score:** 62 / 100 ⚠️

---

## Findings

### ⚠️ High regime dependency
Works well in trending markets (PF 1.8) but fails in chop (PF 0.4). No regime filter means the strategy trades regardless of market conditions — 3 of the last 5 losses happened during low-volatility compression.

### ⚠️ No volatility filter
ATR-based position sizing is static. During high volatility, positions become oversized relative to account equity, amplifying drawdowns.

### ⚠️ No emergency stop mechanism
No circuit breaker. One API failure → infinite retry → duplicate orders → compounded losses.

### ✅ Entry logic is sound
The EMA cross signal has a positive expectancy in trending regimes. With a regime filter, win rate would increase from 42% to an estimated 55-60%.

---

## Recommendations

1. **Add regime detection** — skip signals in CHOPPY markets (estimated +15% PF improvement)
2. **Dynamic position sizing** — reduce exposure when ATR exceeds 2x median
3. **Kill Switch** — auto-pause after 3 consecutive losses or API errors
4. **Execution guard** — deduplicate orders within a 5-minute window

---

## After Fixes (Projected)

| Metric | Before | After |
|--------|--------|-------|
| Win Rate | 42% | 55-60% |
| Max Drawdown | 34% | < 20% |
| Profit Factor | 1.2 | 1.6-1.8 |

---

> *This analysis was generated using the DI Trust Layer — the same framework that runs our own live trading system on OKX.*
>
> [→ Get your strategy checked for free](../../issues/new?title=Health+Check:+%5Bstrategy+name%5D&body=%23%23%20Strategy%0A%0A%23%23%20Market%0A%0A%23%23%20Timeframe%0A%0A%23%23%20Biggest%20Concern%0A)
