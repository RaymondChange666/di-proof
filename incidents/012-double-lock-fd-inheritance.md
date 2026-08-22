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
