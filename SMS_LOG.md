# SMS Cost Log — Chatswood Chase Valet

Daily ClickSend SMS spend, read via the read-only `clicksend-rate` Edge
Function (`{"action":"daily"}` — no SMS is ever sent, just reads ClickSend's
account balance + message history). Rate is currently **$0.0792/segment**
(AUD); most valet texts are 1 segment.

**Bug found & fixed 2026-08-09:** the function was calling ClickSend's
`/sms/history` with no page number, and ClickSend's default is **oldest
message first** — so it had been silently stuck returning the account's
very first messages (early June) no matter how much time passed. Fixed to
walk backward from the last page so "recent" is actually recent.

| Date checked | Balance | Recent burn rate | Notes |
|---|---|---|---|
| 2026-08-09 | $4.93 | ~$0.55/day avg (43-day window, $23.76 total, 288 msgs) — ~9 days of headroom at this rate | Low-balance alert prompted this check. Daily breakdown (last 6 weeks) logged below. Spend is spiky, not flat: 1 Aug ($3.88, 49 msgs) and 28 Jun ($2.77, 35 msgs) were the two biggest days. **Action: top up the ClickSend balance** — at current pace it could run dry within ~1-2 weeks, and a $0 balance means check-in texts silently stop sending (falls back to the in-app share/copy prompt, not a hard failure, but customers stop getting the automatic text). |
| 2026-08-15 | $19.07 | 15 Aug so far: 48 msgs / $3.80 — one of the biggest single days on record, close to 1 Aug's $3.88 high | Balance was topped up since the 08-09 low-balance alert (was $4.93, now $19.07 — comfortable headroom again). Today's spike (48 msgs) is the standout figure; rate unchanged at $0.0792/segment, all recent messages still 1 segment. Function's `daily` response this check only returned 4 recent dates (15, 09, 08, 02 Aug) rather than the full 6-week history seen on 08-09 — window/pagination behavior may have narrowed; not investigated further since balance and rate both look healthy. |

## Daily breakdown (as read 2026-08-09, last ~6 weeks)

| Date | Messages | Cost |
|---|---|---|
| 2026-08-09 | 2 | $0.16 |
| 2026-08-08 | 20 | $1.58 |
| 2026-08-02 | 18 | $1.43 |
| 2026-08-01 | 49 | $3.88 |
| 2026-07-26 | 20 | $1.58 |
| 2026-07-25 | 24 | $1.90 |
| 2026-07-19 | 15 | $1.19 |
| 2026-07-18 | 16 | $1.27 |
| 2026-07-17 | 2 | $0.16 |
| 2026-07-15 | 1 | $0.08 |
| 2026-07-14 | 3 | $0.24 |
| 2026-07-13 | 6 | $0.48 |
| 2026-07-11 | 3 | $0.32 |
| 2026-07-10 | 3 | $0.40 |
| 2026-07-09 | 6 | $0.71 |
| 2026-07-08 | 3 | $0.48 |
| 2026-07-05 | 33 | $2.69 |
| 2026-07-04 | 21 | $1.82 |
| 2026-07-03 | 5 | $0.40 |
| 2026-06-30 | 2 | $0.16 |
| 2026-06-29 | 1 | $0.08 |
| 2026-06-28 | 35 | $2.77 |

*Days not listed had zero messages sent. Missing days between listed ones = no SMS that day.*
