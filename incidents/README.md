# DI System Incidents

## INCIDENT #011 — Trade Ledger Write Failure After Position Close & Missing Position Ownership
**Date:** 2026-08-22 · **Severity:** High · **Status:** Fixed (878ab31 + 3a4057a)

**Symptom:** Trail-stop closed a SOL LONG on 8/21 14:16 then crashed with NameError (`trades_file` not defined) — the post-close ledger write was lost. Same day, a manually-opened App position was erroneously closed by the agent's RSI-reversal exit (-0.84 USDT): `trail_stops` managed every exchange position with no ownership check.

**Root Cause:** (1) `trades_file` was function-local in `recent_pnl()`; `trail_stops` referenced it at module level — the first real close triggered the NameError. (2) Ownership was assumed from exchange state: "position exists" was treated as "I own it".

**Fix:** `878ab31` module-level `trades_file`. `3a4057a` Position Ownership Guard: FILLED → REGISTERED (`open_positions.json`), trail manages only registered positions, exchange>local reconcile every scan, unregistered positions alert-only (never managed), unregister on confirmed close only.

**Lesson:** Funds-state persistence paths need real execution-path validation, not static checks. Exchange state > local state. External/manual trades belong in the reconciliation log, never the strategy ledger.

**Details:** [011-ledger-write-and-ownership.md](011-ledger-write-and-ownership.md)

---

## INCIDENT #010 — A2A Whitelist Drop: Platform Verification Task Discarded
**Date:** 2026-08-20 · **Severity:** High · **Status:** Workaround live; platform-side whitelist gap reported

**Symptom:** OKX support reported ASP #10505 review stuck: the platform's Service 5 verification task (SOL Trend Strategy Backtest, 8 USDT, client #6058) was never received/processed within 1 hour.

**Root Cause:** The `job_asp_selected` envelope DID reach the ASP's XMTP inbox, but the a2a-node daemon dropped it — the verification-flow sender inboxes (`8f816b43…`, `5594be7a…`) are not in the platform's own `system-config` whitelist (verified identical logic in a2a-node 0.2.7 and 0.2.8). Platform-side gap, not a daemon defect.

**Fix:** Manual on-chain processing path (message recovered from XMTP store; apply + deliver txs on chain, backtest executed and delivered honestly); sender inbox IDs reported to OKX support.

**Details:** [010-a2a-whitelist-drop.md](010-a2a-whitelist-drop.md)

---

## INCIDENT #009 — Nine Days of Zero Fills: Order Size Violated Contract Lot Size
**Date:** 2026-08-15 · **Severity:** Critical · **Status:** Fixed (10e5e9c + ae27754)

**Symptom:** hrscan v6 live strategy ran 9 days (Aug 6-15) with zero filled orders. All 97 trade records were rejected orders. SUSHI logged "All operations failed" 82 times; BNB entries printed as success while no order existed on OKX.

**Root Cause:** Three stacked execution bugs: (1) order size not floored to contract lotSz — SUSHI/WIF lotSz=1 rejects fractional sizes, SOL lotSz=0.01 masked the bug; (2) OKX rejection reason lives in data[0].sMsg while top-level msg is empty — code printed "OK" for rejections (retroactively reframes INCIDENT #007's "12 successful BNB orders" as 12 mislabeled rejections); (3) trade log written unconditionally — rejected orders became phantom OPEN positions.

**Fix:** 10e5e9c: per-coin lot_sz + size floor, sMsg passthrough, success-only trade log, wrapper py_compile gate restored. ae27754: reconciler skips rejected records; 97 phantom entries backfilled rejected=True; ledger reset to honest zero.

**Lesson:** Log "success" ≠ order exists. orders-history is ground truth for fills. A system can look perfectly healthy while the execution layer is 100% dead — inspection must cross-check trade records against orders-history.

**Details:** [009-zero-fill-execution-bugs.md](009-zero-fill-execution-bugs.md)

---

## INCIDENT #008 — Silent Daemon Failure & Monitoring Blind Spots
**Date:** 2026-08-15 · **Severity:** Critical · **Status:** Fixed

**Symptom:** Factory shadow bot offline 9 days undetected; heartbeat showed stale state as live. Second finding: breakout20 TIME_STOP unreachable (bars_held hardcoded 0) — worker alive but position could never time out. Third: control_api monitoring a retired system (config drift).

**Root Cause:** (1) Malformed systemd unit (semicolon-packed directives) — every restart failed silently. (2) Execution-correctness bug: exit call hardcoded bars_held=0. (3) Monitor target not repointed after system replacement.

**Fix:** Controlled unit rewrite (backup + diff + reload), TIME_STOP derives real bars held from entry time (6/6 tests pass), di_pulse upgraded to full-chain liveness (HK 2/11 → 11/11 components via monitor_targets.yaml), shadow_heartbeat bridged to control_api (health 0 → 100).

**Lesson:** Process alive ≠ behavior correct. Monitoring must verify output freshness, not file existence.

**Details:** [008-silent-daemon-failure.md](008-silent-daemon-failure.md)

---


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
