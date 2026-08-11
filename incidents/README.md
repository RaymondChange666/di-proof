# DI System Incidents

> *Every failure is public. Every fix is proven. This is the Trust Layer.*

---

## INCIDENT #007 — Minimum Equity Gate Blocks All Trading
**Date:** 2026-08-11 · **Severity:** Critical · **Status:** Fixed (d53951f)

**Symptom:** Every scan printed `SKIP: equity $70.4 < $500 minimum`. Strategy completely stopped for ~30 minutes.

**Root Cause:** `get_eq()` read `det['eq']` filtered by `ccy == 'USDT'`, returning USDT balance ($70) instead of `totalEq` ($525). The $500 minimum equity gate then blocked all scans.

**Fix:** Changed `get_eq()` to read `d['totalEq']` from the balance response, which correctly returns total account value across all currencies.

**Lesson:** Per-currency equity != total equity in multi-currency accounts. Always use the aggregate field for gating decisions.

---

## INCIDENT #006 — SUSHI Order Rejection Loop (×10)
**Date:** 2026-08-10 · **Severity:** High · **Status:** Fixed (471b186)

**Symptom:** SUSHI-USDT-SWAP LONG triggered 10 times in 30 minutes (18:51-19:19 UTC). Every order returned "All operations failed" from OKX.

**Root Cause:** OKX rejected the orders (transient issue — same orders succeeded in later testing). Since no position was created, `get_positions()` correctly returned empty, and the next minute's scan triggered again. Unlike INCIDENT #005, this was NOT a positions API failure — it was a legitimate order rejection followed by an infinite retry loop.

**Fix:** Added 30-minute signal cooldown per symbol+direction after a failed order. Cooldown state persisted to `hrscan_cooldown.json`.

**Lesson:** Order rejection ≠ position detection failure. Two different bugs with similar symptoms require two different fixes. Every failure path needs its own guard.

---

## INCIDENT #005 — BNB Duplicate Orders (×12)
**Date:** 2026-08-06 · **Severity:** Critical · **Status:** Fixed (f5e4b45)

**Symptom:** BNB-USDT-SWAP LONG opened 12 times in 12 minutes (07:34-07:45 UTC). All orders succeeded, creating 12 duplicate positions.

**Root Cause:** `get_positions()` had a bare `except: pass` that silently returned `{}` on ANY API failure (network error, rate limit, non-zero response code). The position dedup check then saw zero positions and allowed the same signal to execute repeatedly.

**Fix:** Added 3-retry loop with 1-second intervals. On persistent failure, returns `None` instead of `{}`, causing the entire scan to skip (fail-safe). Each failed attempt now logs the error code and message.

**Lesson:** Silent failure modes are the most dangerous bugs in autonomous systems. Every API call needs explicit error handling. `except: pass` is a liability in production trading code.

---

## INCIDENT #004 — Heartbeat Writer Silent Death
**Date:** 2026-08-08 · **Severity:** Medium · **Status:** Fixed

**Symptom:** `di_heartbeat.json` stopped updating on Aug 5. Went undetected for 3 days. No alert triggered.

**Root Cause:** `di_pulse.py` was killed during system cleanup but not re-added to crontab. The old daemon monitoring list was stale (monitoring v5 daemons that were already killed).

**Fix:** Updated daemon list to v6.1 reality. Added `di_pulse.py` to crontab. Added hrscan v6 cron status check.

**Lesson:** Monitoring infrastructure needs its own monitoring. A dead heartbeat writer is a silent failure — the system appears healthy because no alert fires.

---

## INCIDENT #003 — verify_push Hangs on GitHub CDN
**Date:** 2026-08-08 · **Severity:** Low · **Status:** Fixed

**Symptom:** `verify_push.sh` never completed. Log file always empty. Daily report push verification was blind.

**Root Cause:** Script curled `raw.githubusercontent.com` from AliCloud (China). GFW blocked the request, causing infinite hang. Cron timeout killed it before any output.

**Fix:** Replaced remote CDN check with local status file (`/root/di-proof-repo/.push_status`). `gen_artifact.py` writes push result; `verify_push.sh` reads it locally.

**Lesson:** Don't assume network reachability. Design verification to work within known network constraints.
