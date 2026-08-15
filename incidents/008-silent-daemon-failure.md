# INCIDENT #008 — Silent Daemon Failure & Monitoring Blind Spots

**Date:** 2026-08-15 · **Severity:** Critical · **Status:** Fixed
**Detection:** DI Trust Layer full-chain liveness audit (Phase 0.5)
**Category:** Reliability / Observability

---

## Event 1: Factory daemon offline 9 days (undetected)

**Symptom:** Factory v1 shadow bot stopped writing logs at 08:04 CST on 2026-08-06.
Heartbeat kept reporting "3 active positions" for 9 days. Positions were stale
state.json data, not live positions.

**Root Cause:** Malformed systemd unit file. Two lines packed multiple
directives with semicolons:

```
Type=simple; User=root; WorkingDirectory=/root/bot
Restart=always; RestartSec=10
```

systemd could not parse these lines ("Failed to parse service type"),
so every restart attempt since 2026-08-06 failed. The daemon could never
come back up.

**Impact:** No live execution for 9 days. State freshness invalid.
Monitoring showed false health (stale state.json read as live).

**Fix (controlled change, backup + diff + verification):**
- Backup: `factory.service.bak.incident007`
- Unit rewritten to standard multi-line format + `StartLimitBurst=5`
  (crash-loop protection)
- `systemctl daemon-reload` → parse errors gone → service loaded

**Recovery validation (controlled restart, 10-min observation):**
- Service active, 0 restarts, 0 tracebacks
- 3 stale positions auto-reconciled on restart — all TP wins
  (SOL +$0.137, ETH +$0.289, BNB +$0.122), equity $517.6 → $518.2
- Marked `incident_recovery_reconciliation` — excluded from Sample V2
- Normal signal cycle resumed (WIF SHORT opened on RSI=77)

---

## Event 2: breakout20 TIME_STOP unreachable (bars_held=0)

**Symptom:** BTC 4H shadow worker silent since 2026-08-12 00:36. Process
alive, CPU normal, sleeping on schedule — textbook "alive but behavior
diverged from design".

**Root Cause:** `shadow_worker_v3.py` called `check_exit(price, 0, MAX_BAR)`
with `bars_held` hardcoded to 0. TIME_STOP condition is
`bars_held >= max_bars`, so `0 >= 100` is never true. A position opened
2026-08-12 (BTC SHORT @63584.7, SL 64544.74, TP 61664.61) sat inside the
SL/TP dead zone (BTC ~63005) and could never time out. The position-holding
branch silently `continue`s, so the worker produced zero output.

**Impact:** Position lifetime protection disabled. Risk control (max hold
100 bars) existed in design (`BREAKOUT_CONFIG.max_hold_bars: 100`) but was
unreachable in code.

**Fix:** Derive real bars held from entry time:

```python
TIMEFRAME_MINUTES = {"1m": 1, ..., "4h": 240}.get(TIMEFRAME.lower(), 60)

def bars_held_for(trade) -> int:
    return int((time.time() - trade.entry_time.timestamp())
               // (TIMEFRAME_MINUTES * 60))

bars_held = bars_held_for(st)
reason = st.check_exit(price, bars_held, MAX_BAR)
```

**Tests (all pass):**
- A: TIME_STOP fires at bars == max_bars
- B: holds at bars < max_bars
- C1-C3: SL/TP behavior unaffected
- D: bars_held_for time math correct (~100 bars for 401h on 4H)

---

## Event 3: Configuration drift (control_api monitoring stale target)

**Symptom:** control_api `/health/shadow` reported `critical, health_score: 0`
permanently — "Heartbeat file missing".

**Root Cause:** control_api's health adapter read
`/root/bot/runtime/shadow_heartbeat.json`, written by the OLD shadow system
(retired 2026-08-05). The new system (shadow_worker_v3) never writes that
file. The system upgraded; the monitor did not follow.

**Fix:** di_pulse v2 (HK) now bridges a fresh `shadow_heartbeat.json` from
live liveness checks every minute. control_api health score recovered:
0 (critical) → 73 (degraded, honest) → 100 (healthy) after TIME_STOP fix.

---

## Monitoring upgrade (prevention)

Before this incident, HK di_pulse monitored 2 of 11 components. Now:

- **Aliyun di_pulse:** 5 systemd daemons + factory log-freshness alert
  (`log_age_s > 1800` → ⚠️)
- **HK di_pulse v2:** 11 components via `/root/monitor_targets.yaml`
  (systemd / cron / dir / http check types), per-component `max_age`
  tuned to each agent's lifecycle pattern (continuous / hourly /
  event-driven)
- False-alarm calibration: paper_hrscan / shadow_trend / shadow_breakout
  only print on the hour (`tm_min == 0`) by design → max_age 3600s

---

## Lessons

1. **Stale state ≠ healthy.** Monitoring must check process liveness AND
   output freshness. A file's existence proves nothing.
2. **Restart policy is part of correctness.** A daemon that can't restart
   after a crash is a latent single point of failure. systemd unit syntax
   errors are silent killers.
3. **Time-based exits must be reachable.** A risk control that exists in
   config but not in code is a false sense of safety. Unit-test exit paths.
4. **Monitors must follow system evolution.** When a system is replaced,
   its monitors must be repointed — or they report false alarms (or worse,
   false health).
5. **"Process alive" is not "behavior correct".** The highest-value
   detection is finding a system that is alive but has diverged from
   design.

---

*Fix commits and backups: `/root/incidents/007/` on Aliyun (evidence
freeze: raw logs, state, unit file) · `factory.service.bak.incident007` ·
`di_pulse.py.bak.incident007` (both hosts) · `shadow_worker_v3.py.bak.bars_held`
(HK)*
