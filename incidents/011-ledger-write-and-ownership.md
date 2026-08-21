# INCIDENT #011 — Trade Ledger Write Failure After Position Close & Missing Position Ownership

**Date:** 2026-08-22 · **Severity:** High · **Status:** Fixed (878ab31 + 3a4057a)

## Symptom

On 2026-08-21 14:16 CST, after the trail-stop closed a SOL LONG position
(1H RSI reversal exit), the hrscan process crashed with
`NameError: name 'trades_file' is not defined`. The close executed on the
exchange, but the local trade ledger write failed, leaving the
reconciliation pipeline with zero records for that day.

Same day, three exchange fills (14:15 buy 7, 14:21 sell 7, 15:15 limit buy 7)
had no hrscan log lines at all — they were opened outside the agent.

## Root Cause

1. `trades_file` was defined only as a local variable inside `recent_pnl()`.
   The `trail_stops()` exit-logging block (added in 49e5013) referenced it
   as a module-level name. The reference was never exercised until the first
   real RSI-reversal close on 8/21 — it crashed exactly when the ledger
   record mattered most.
2. Ownership assumption: `trail_stops()` managed ALL positions returned by
   the exchange positions API with no ownership check. A manually-opened
   App position (buy 7 SOL @ 90.80, 100x) was treated as hrscan-owned and
   closed by the RSI-reversal exit logic (sell 7 @ 90.68, -0.84 USDT).

## Impact

- One real close path crashed post-close logging (execution itself on the
  exchange was unchanged).
- Local ledger integrity broken for 8/21 — reconcile/daily-report pipeline
  consumed empty data.
- One external (manual) position erroneously managed and closed by the
  agent: -0.84 USDT realized against the manual trader.

### External trades on 2026-08-21 (NOT backfilled into strategy ledger — per 总裁裁决)

| ts (CST) | instId | side | sz | px | pnl (USDT) | source |
|---|---|---|---|---|---|---|
| 14:15:07 | SOL-USDT-SWAP | buy | 7 | 90.80 | 0 | manual app |
| 14:16:02 | SOL-USDT-SWAP | sell | 7 | 90.68 | -0.84 | hrscan erroneous close of manual position |
| 14:21:53 | SOL-USDT-SWAP | sell | 7 | 90.47 | 0 | manual app |
| 15:15:25 | SOL-USDT-SWAP | buy | 7 | 90.98 | -3.57 | manual app |

Recorded in `/root/bot/external_trades.jsonl` (reconciliation log),
NOT in the strategy performance ledger.

## Fix

- `878ab31`: module-level `trades_file` definition (+3 lines).
- `3a4057a`: Position Ownership Guard:
  - ORDER FILLED → POSITION REGISTERED (`open_positions.json`)
  - `trail_stops()` manages ONLY registered positions
  - reconcile every scan: exchange state > local state (drop closed,
    revoke side-flipped, adopt + alert external size changes)
  - unregistered exchange positions: alert once (`UNMANAGED_POSITION`
    data event), NEVER manage
  - RSI-reversal close unregisters only on confirmed success
    (code==0, no sMsg)

## Verification

- `py_compile` clean
- AST assertion: module-level definition, no local shadow
- Kill Switch real execution: exit 0, no traceback
- Natural cron execution: no new tracebacks
- 10/10 unit tests (`test_ownership.py`, AST-extracted real functions +
  mocked OKX):
  - T1 owned position managed + unregistered after close
  - T2 foreign position never managed + alert
  - T3 closed position registration dropped
  - T4 restart recovers ownership from file
  - T5 side flip revokes ownership, never manages
  - T6 external size change adopted + alert

## Lesson

Any persistent path touching funds state must be validated on the real
execution path, not just with static/startup checks — a reference that is
only exercised on the first real close crashes exactly when the record
matters most. Exchange state always wins over local state, and
"a position exists on my account" must never be treated as "I own this
position". Manual/external trades belong in a reconciliation log, never in
the strategy performance ledger.
