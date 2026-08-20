# INCIDENT #010 — A2A Whitelist Drop: Platform Verification Task Discarded

**Date:** 2026-08-20 · **Severity:** High · **Status:** Workaround live (manual on-chain processing); platform-side whitelist gap reported

## Symptom

OKX support reported ASP #10505 review stuck at "Listing under review":
the platform sent a Service 5 verification task (Job
`0xf9f5f318a228683f7f941dad4617c71cb52ac3cfacc5e0d0656bea3587f63b86`,
"SOL Trend Strategy Backtest", 8 USDT, client agent #6058) and never saw
the ASP receive or process it within 1 hour.

## Discovery

Support reply on 2026-08-20 (~15:45 CST) with the Job ID. Investigation of
`~/.okx-agent-task/logs/listener.log` and the XMTP SQLite store found:

- 13:54:57 CST — the `job_asp_selected` envelope DID arrive in the ASP's
  XMTP inbox (sender inbox `8f816b43...`, message intact in `group_messages`).
- The live XMTP stream was throwing repeated `AgentStreamingError`
  (322 occurrences that day), so the message was not handled in real time.
- 15:47:03 CST — periodic client recycle triggered offline replay; the
  message was finally picked up (`replayed=1`) but then dropped:
  `DM sender not in system whitelist, dropping:
  senderInboxId=8f816b43...`
- 15:54:00 CST — the follow-up `job_accepted` envelope (new sender
  `5594be7a...`) was dropped the same way.

## Root Cause

The a2a-node daemon only accepts DMs from inboxes listed in the platform's
`system-config` (`onchainos agent system-config`). At drop time the list
contained exactly two accounts:
`91ed1317...` (type=system) and `f9255043...` (type=security).
The inboxes used by the platform's verification/task flow
(`8f816b43...`, `5594be7a...`) are not in the platform's own config —
freshly re-fetched at 15:47 before the drop. Verified identical
whitelist logic in a2a-node 0.2.7 and 0.2.8, so a daemon upgrade does
not change the behavior; the gap is platform-side.

Secondary (non-blocking): `provider=hermes gateway check failed
(plugin_yaml_missing)` since 2026-08-04 — the daemon checks
`$HERMES_HOME/plugins/platforms/okx-a2a/plugin.yaml` with default
`~/.hermes`, but this machine runs the `crypto-trader` profile. This only
affects heartbeat gating (heartbeat is sent anyway); message dispatch to
Hermes worked independently (proven by 08-19 AI sessions).

## Fix / Workaround

- Manual on-chain processing (message content recovered from XMTP store):
  1. `onchainos agent next-action` (event=job_asp_selected) → playbook
  2. `onchainos agent apply` → tx `0x03b53b90...` (16:00 CST)
  3. `user-notify` on job_accepted
  4. Backtest executed locally: OKX history-candles, 91,968 × 15m + 5,748 × 4h
     bars, honest result (see deliverable; strategy -12.44% vs buy&hold
     -26.37%, MDD -15.0% vs -79.5%)
  5. `onchainos agent deliver` (text path, GFW-safe) → tx `0x48437503...`
- Reported to OKX support with sender inbox IDs; asked platform to add
  verification-flow inboxes to `system-config`.
- Deferred (stable-first): daemon restart with `HERMES_HOME` env fix and
  a2a-node 0.2.8 upgrade — scheduled after review completes.

## Lessons

1. **On-chain state ≠ daemon log.** The message can be fully received and
   still never processed — watch the dispatch path end-to-end, not just the
   listener.
2. **The XMTP store is ground truth.** The dropped envelope was fully
   recoverable from SQLite; that recovery is what made manual processing
   possible. Never treat a dropped message as lost.
3. **Platform system-config is authoritative and not under our control.**
   Detection must cover "message received but dropped" — add a
   `DM sender not in system whitelist` alert to exchange-side checks.
