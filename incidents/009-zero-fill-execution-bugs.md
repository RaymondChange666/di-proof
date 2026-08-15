# INCIDENT #009 — Nine Days of Zero Fills: Order Size Violated Contract Lot Size

**Date:** 2026-08-15 · **Severity:** Critical · **Status:** Fixed (10e5e9c + ae27754)

## Symptom

hrscan v6 live strategy ran from Aug 6 to Aug 15 without a single filled order.
All 97 records in `hrscan_trades.jsonl` were rejected orders. SUSHI logged
"All operations failed" repeatedly (82 times). BNB entries printed as success
while no order ever existed on OKX.

## Discovery

Routine inspection on Aug 15: heartbeat showed `pos=[]` (empty) while the trade
ledger showed 4 phantom OPEN positions. Cross-checking `trades.jsonl` against
OKX `orders-history` API revealed the account had only 5 terminal orders in 3
months (2 manual test orders on Aug 8, 3 test orders on Aug 11) — zero strategy
fills.

## Root Cause — three stacked execution bugs

1. **Order size not floored to contract `lotSz`.**
   SUSHI/WIF contracts have `lotSz=1` (integer contracts only). The code sent
   fractional sizes (e.g. `sz=158.86`), which OKX always rejects.
   SOL has `lotSz=0.01`, so 2-decimal rounding happened to be legal — masking
   the bug for 9 days.

2. **OKX rejection reason lives in `data[0].sMsg`; top-level `msg` is empty.**
   The code read `r.get('msg')` — empty on rejection — and printed "OK".
   This also retroactively reframes INCIDENT #007: the "12 successful BNB
   orders" were 12 rejections mislabeled as success. `msg=""` ≠ success.

3. **Trade log written unconditionally.**
   Rejected orders were appended to `trades.jsonl`, and the reconciler turned
   them into phantom OPEN positions in the ledger.

## Fix

- `10e5e9c` — per-coin `lot_sz` in COINS config, size floored to lot multiple;
  rejection `sMsg` passthrough; trade log written only on success; restored
  wrapper `py_compile` gate (had reverted to a tautological `[ 0 -eq 0 ]`).
- `ae27754` — reconciler skips `rejected` records; all 97 phantom entries
  backfilled `rejected: true`. Ledger reset to honest zero.

## Impact

Phase 1 evidence accumulation restarts from the next real fill. The 9 days of
"trading" produced no PnL sample. Equity untouched ($525).

## Lesson

**Log "success" ≠ order exists.** `orders-history` is the ground truth for
fills. A trading system can look perfectly healthy — scans run, heartbeats
fresh, logs clean — while the execution layer is 100% dead. Inspection must
periodically cross-check trade records against `orders-history`.

---
*Every failure is public. Every fix is proven. This is the Trust Layer.*
