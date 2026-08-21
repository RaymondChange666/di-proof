# P0 Reliability Sweep — 2026-08-22

Third-party adversarial audit of the production trading system, followed by
a full fix cycle. Public record of what was found, what was fixed, and how
it was verified.

## Context

Production strategy alpha is frozen. Before building the competition branch,
the live system was handed to an independent lead engineer (Claude Sonnet 5)
as a sanitized audit bundle (source + tests + incidents + git history).
The engineer was asked to find what we had not found ourselves.

## Phase 1 — Independent audit: 5 P0s confirmed real

Every claim was verified against the code before fixing (no blind trust).

| # | Finding | Severity |
|---|---|---|
| P0-1 | No single-instance lock — overlapping cron ticks could race duplicate orders | P0 |
| P0-2 | Failed close still wrote an exit record — ghost exit in ledger | P0 |
| P0-3 | Ledger pairing matched every entry to the SAME exit — PnL/PF/Sharpe inflated | P0 |
| P0-4 | Non-atomic state writes; corrupted state loaded as `{}` (empty = trading continues) | P0 |
| P0-5 | No unknown-execution handling — network failure on order placement left state ambiguous | P0 |

### Fixes (one commit each)

- `70fbab5` P0-1: flock single-instance guard in run_hrscan.sh (4-scenario verified)
- `047350b` P0-2: exit records only on confirmed close; CLOSE_FAILED events; exit gains sz field
- `4cb6219` P0-3: FIFO size-aware pairing; pro-rata pnl split; UNMATCHED flagged
- `f1e53f7` P0-4: tmp+fsync+rename atomic writes; corrupted state → None → no new orders, circuit blocked
- `c8611bd` P0-5: clOrdId on every order; network exception → query by clOrdId; pending registry + adoption

## Phase 2 — Adversarial review of our own patches: 9 findings, all verified real

The same engineer then attacked the fixes. All 9 findings were confirmed in code.

| # | Finding | Verdict |
|---|---|---|
| P0-5-a | Pending adoption matched only instId — could adopt and close an unrelated manual position (INCIDENT #011 class bug, new path) | P0 — fixed |
| P0-2-a | Close path had NO network-exception handling — P0-5 covered opens only | P0 — fixed |
| P0-3-a | Split entries produced duplicate trade_ids | P1 — fixed |
| P0-5-b | Pending expiry dropped silently, no alert | P1 — fixed |
| P0-3-b | UNMATCHED ledger records had no monitoring exit | P1 — fixed |
| P0-2-b | Failed closes retried every tick with no backoff | P1 — fixed |
| P0-1-a | Lock lived only in the shell wrapper — manual invocations unguarded | P1 — fixed |
| P0-4-a | Test assertion checked a misspelled path (`test_owned._json.tmp`) — permanently true, tested nothing | P2 — fixed |
| P0-1-b | No watchdog — a stalled tick would silently halt trading | P2 — fixed |

### Fixes

- `01f3203` Adoption hardened: side must match pending sig AND clOrdId look-up
  must confirm `filled`; close path gained clOrdId query recovery + 30-min
  close cooldown; PENDING_EXPIRED alerts; ADOPTED_NO_SLTP critical alert.
  3 adversarial test cases added (side-mismatch rejection, partial-fill
  adoption, close-recovery without duplicate orders).
- `aac5f4e` Unique trade_id per split (per-key sequence); UNMATCHED_EXIT events.
- `b54bfd0` Python-level fcntl lock (same lock file, guards manual runs).
- `4022f47` Watchdog: wrapper writes last_success heartbeat; di_pulse alerts
  when no successful tick for 180s.

## Verification

- 37/37 ownership + execution regression tests (AST-extracted real functions, mocked OKX)
- 14/14 FIFO reconcile tests (multi-entry/multi-exit, pro-rata, legacy records, uniqueness)
- 7/7 risk_guard fail-safe tests (corrupted state → circuit blocked)
- Kill-switch dry runs, py_compile, natural cron execution — all clean

## Lessons

1. **"Position exists" is never "position is mine."** The ownership invariant
   must hold on every path — including recovery paths. An adoption shortcut
   reintroduced the exact INCIDENT #011 bug in a new form.
2. **Audit the auditor's fixes too.** The adversarial round found 9 real issues
   in patches that had passed our own tests — including one test that was
   silently testing nothing (misspelled path, permanent pass).
3. **Network ambiguity is a first-class state**, not an error to catch and
   forget: clOrdId query + pending registry + expiry alerts form one lifecycle.
4. **Exchange state > local state**, and unverifiable local state must fail
   safe (no new orders), never fail open.
5. **Watchdogs need heartbeats.** A lock without a watchdog trades duplicate
   orders for silent death.
