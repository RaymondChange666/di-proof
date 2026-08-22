# INCIDENT #012 — Double-Lock fd Inheritance: hrscan Silently Dead for 16 Hours

**Date:** 2026-08-22 · **Severity:** High · **Status:** Fixed (b7f32c0 + b3e0cca)

## Symptom

From 03:02 CST, every cron tick logged `hrscan already running (python-level lock) - exiting`.
No hrscan process existed (`ps` empty), no holder of `/tmp/hrscan_v5.lock` (`lsof`/`fuser` empty),
and a manual `flock -n` succeeded — yet every tick self-blocked. Meanwhile
`last_success.txt` refreshed every minute, so the watchdog reported healthy.
The live strategy silently stopped scanning for 16 hours. No positions were at risk
(flat the whole window); the cost was missed signal coverage.

## Timeline

- 02:29 commit 70fbab5 (P0-1 sweep): shell wrapper gains `exec 9>lock; flock -n 9` single-instance guard
- 03:02 commit b54bfd0 (adversarial review R7): python gains `fcntl.flock` on the SAME lock file
- 03:02:14 first `already running (python-level lock)` line — exact match with commit time
- 21:15 fix deployed; lock conflict gone from 21:16 onward

## Root Cause

Double single-instance lock with fd inheritance conflict:
1. wrapper `exec 9>"$LOCK"` then `flock -n 9` acquires the lock on fd 9
2. wrapper launches python as a child — **fd 9 is inherited**
3. python `open(lock)` gets a NEW fd and calls `fcntl.flock(LOCK_EX|LOCK_NB)`
4. flock(2) locks conflict across open file descriptions, even within the same process —
   the inherited fd 9 still holds the lock → python always raises OSError → "already running" → exit

Amplifier: python exited with code 0 on lock failure, so the wrapper's `&& date +%s >
last_success.txt` fired — the P1 watchdog (whose whole purpose was to catch exactly this
class of stall) reported false health. The second lock was added by adversarial review R7;
nobody ran a single real execution after the patch landed — py_compile green ≠ working.

## Fix

1. b7f32c0 — wrapper releases its lock before launching python: `flock -u 9`.
   The python-level lock becomes the sole execution guard (it covers manual invocations,
   which was R7's original purpose).
2. b3e0cca — python lock-failure exit code 0 → 1, so `last_success` only records real
   scan completions. A genuine concurrent invocation now surfaces in the watchdog.

Verification: `already running` lines stopped; manual `python3.11 hrscan_v5.py` runs the
full scan path; OKX positions API returns code 0; `last_success` updates on real ticks;
`exchange_side_alerts.json` ok=true.

## Lesson

1. **Two-layer locks must be tested for fd inheritance.** Adding a second guard and
   verifying "conflict path works" is not enough — the happy path (acquire both locks in
   sequence) must be exercised with a real execution before the patch lands.
2. **Watchdog writers must bind to real success semantics.** A heartbeat that fires on
   failed-fast exits is a false-health generator, not a monitor.
3. **"py_compile OK" is not a deployment test.** After any patch that touches the
   startup/guard path, run the actual binary once and read its output.

**Details:** this file.


## Post-Incident Verification & Governance Addendum (2026-08-22)

### 1. Timeline precision
First `already running (python-level lock)` line: 03:02:01 — 13s before commit
b54bfd0 (03:02:14). The file edit landed in the working tree ahead of the commit;
mechanism unchanged.

### 2. Missed-signal replay: 0 missed
Per-minute replay of the outage window (03:00:01-21:16:59 CST) with the exact live
logic (calc_rsi cumulative variant, 60x15m closes + partial candle close, 20x1H
closes, ma5/prev_close conditions, tf_confirm 1H<40-LONG / 1H>60-SHORT block),
across all three sentiment-threshold variants (default / bull-boost / bear-boost):
- SOL:   RSI15 range 47.8-77.7 -> 0 entry signals passing tf_confirm
- WIF:   RSI15 range 49.3-71.5 -> 0
- SUSHI: RSI15 range 46.5-72.3 -> 0
Equity gate passed the whole window (totalEq 605-681 >= $500, 751 heartbeat
samples); positions flat throughout. Conclusion: zero missed real trade
opportunities — the cost of this incident is signal coverage only, not missed
executions.

### 3. Governance findings
Three Hermes sessions operated on production concurrently on 2026-08-22:
- Session A (CLI): deployed the P0/P1 sweep 02:31-03:02 including b54bfd0 (the
  accident cause), performed zero post-deploy verification, and moved on unaware.
- Session B (CLI): detected the outage at 19:20, deployed the fix at 21:15:57
  (commits b7f32c0 + b3e0cca at 21:21), published this incident. NOTE: earlier
  reports misattributed these commits to "another session" — corrected after
  git-log verification. A session must verify its own history before attributing
  actions to others.
- Session C (desktop): independently re-diagnosed with an incorrect mechanism
  (stuck process), produced a v4 patch (timeout+SIGKILL hardening) routed through
  T2 separately from this incident.

Adopted rule: any session touching /root/bot must (a) `git log -3` + read the
latest incident first, and (b) verify the next real execution cycle's log after
deploy — "deployed" means verified, not just written.

### 4. "Who killed the stuck process?" — dissolved
There was no stuck process. ps/lsof/fuser were empty mid-outage (19:20-19:21 CST);
the lock was re-acquired and re-collided every minute by the fd-inheritance
mechanism itself. Recovery at 21:16 was the fix deploy, not a kill. No unknown
killer to record.


## v4-final hardening deployed (2026-08-23 03:56 CST, T2 round-3 + president line-by-line review)

Two-file deploy (python first, wrapper second, md5-verified, backups .bak.v4final):
1. wrapper: `timeout --kill-after=15 300` + `python3.11 -u`; last_success write REMOVED
   from wrapper (moved into python); `flock -u 9` preserved with INVARIANT comment.
2. python: in-python atomic `touch_last_success()` (tmp+fsync+rename); GATE_SKIP x2
   (circuit_breaker/min_equity - expected pauses, heartbeat continues) vs GATE_ERROR x3
   (positions_api_failure/ownership_corrupted/cooldown_corrupted - real faults, no
   heartbeat -> R9 watchdog alarms within 180s); per-tick `tick OK` heartbeat + flush.
3. President review correction: cooldown_corrupted initially GATE_SKIP - reclassified
   GATE_ERROR (state corruption is a fault, not a pause; the branch itself writes
   STATE_CORRUPTION events).

Post-deploy verification (03:57:03 tick): tick OK printed with correct local time,
zero python-level lock collisions, last_success mtime == tick OK timestamp (python
internal write path confirmed), zero GATE_ERROR lines.
