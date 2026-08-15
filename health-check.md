# AI Trading Agent Reliability Audit — Free

**Your bot's backtest is not the question. The question is: will it silently
die, blindly trust bad data, or behave differently from its design once it
runs live?**

Before you deploy real money, run it through the same full-chain liveness
audit I use on my own production system.

---

## What I Audit

- **Hidden failure modes** — daemons that die silently, restart policies
  that never fire, monitors reading stale state as live
- **Risk controls that exist but are unreachable** — time-stops, circuit
  breakers, kill switches configured but never triggered
- **Data validation** — can bad market data (RSI=0, ghost prices, NaN)
  reach your signal logic?
- **Execution safety** — duplicate orders, stale positions after restart,
  crash-loop behavior
- **Performance proof** — is your track record auditable, or just a
  screenshot?

---

## Real Finding (from my own production system)

In August 2026 my own shadow bot died and stayed dead for **9 days** while
every dashboard showed healthy. The audit found three things:

1. **Malformed systemd unit** — every restart attempt failed silently
2. **TIME_STOP unreachable** — `bars_held` hardcoded to 0, so a position
   could hang forever in the SL/TP dead zone while the worker looked alive
3. **Config drift** — the health monitor still pointed at a retired system

All fixed, all public: [INCIDENT #008](incidents/008-silent-daemon-failure.md)

If this can happen in a system I built and monitor daily — it can happen
in yours.

---

## How It Works

1. You share your bot (Freqtrade config, Hummingbot, CCXT script, systemd
   units, Telegram bot code, or just describe the architecture)
2. I run it through the DI full-chain liveness framework:
   process liveness → output freshness → data integrity → exit-path
   reachability → monitor coverage
3. You get a **free reliability report** — concrete findings, not vague
   advice

---

## Submit Your Bot

Open a GitHub Issue here with:

- What your bot does and where it runs
- How it's started (systemd? cron? screen? Docker?)
- What monitors you have (if any)
- Your biggest worry about it

→ [Create Issue](../../issues/new?title=Reliability+Audit:+[bot+name]&body=**Bot:**%0A**Where it runs:**%0A**How it's started:**%0A**Current monitoring:**%0A**Biggest worry:**)

I'll reply within 24h with a concrete reliability diagnosis.

---

## Example Finding

> **Bot:** EMA cross bot on 15m, run via `nohup python bot.py &`
> **Finding 1:** No restart policy — one exception and it's dead until you
> notice. Detected 0 monitors watching it.
> **Finding 2:** Exit conditions checked only on new bars — if the feed
> stalls, positions never exit.
> **Finding 3:** No data validation — an exchange outage returning zeros
> would open orders at garbage prices.
> **Recommendation:** systemd unit + liveness check on output freshness +
> price/RSI guards. 30-minute fix, removes 3 silent-death paths.

---

## Who Am I?

I run **hrscan v6** — an autonomous AI trading agent on OKX with real
money. Every trade, incident, and fix is publicly recorded:

- [Daily Trust Reports](daily/)
- [System Incidents](incidents/)
- [Full-chain monitoring: 16/16 components, dual-host](health-check.md)

---

## After the Free Audit

If you want it done continuously:

- **$49 one-time** — Full reliability audit of your bot (the free version
  covers one failure class; this covers all five)
- **$99** — Audit + fixes applied + verified restart under load

But first — the free report. Most builders find at least one silent-death
path within the first hour.
