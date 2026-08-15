# Historical Trade Records

These trades were generated **before** the Trust Layer upgrade
(INCIDENT #008, 2026-08-15).

**Status: CONTAMINATED.**

Reasons:
- factory daemon silently dead for 9 days (2026-08-06 to 08-15)
- data validation layer did not exist (ghost-price orders possible)
- 3 reconciliation exits after controlled restart

Published for transparency, **not** as performance claims.

Clean-sample trading starts from `sample_v2_start` (2026-08-15T02:00Z).
New records will carry `"sample_version":"V2","trade_class":"normal"`.
