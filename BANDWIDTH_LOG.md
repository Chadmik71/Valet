# Bandwidth (Egress) Log — chatswood-valet

Daily Supabase egress for **this app only** (chatswood-valet project), read from the
Supabase dashboard, Settings → Usage, filtered to project = chatswood-valet.
(Org-wide numbers include a second, unrelated project — `scone-thai-massage` — so the
org-level total is NOT the right number to quote for this app.)

Org is on the **Pro plan**: 250 GB/month included egress, pooled across all projects
in the org. Billing cycle: 06th–06th.

| Date (Sydney) | Egress used (this app, cumulative in cycle) | Notes |
|---|---|---|
| 2026-08-07 (morning) | 0.00 GB | Billing cycle reset 06 Aug 2026. Per-project egress chart shows "No data in period" — Supabase says this can take up to 24h to populate after a cycle reset. Baseline reading only, not a real daily figure yet. |
| 2026-08-07 (re-check, ~1hr later) | 0.00 GB | Still "No data in period" on the per-project daily chart. Not yet reflecting live traffic — needs the full 24h window to populate. |

## Code audit — 2026-08-07: confirmed no regression of the July egress bug

Re-read `index_v2.html` end-to-end for anything that could re-blow the egress budget,
prompted by the customer bandwidth question. Findings:

- The 60s poll (`reconcileFromCloud('poll')`, index_v2.html:14163) still takes the
  cheap **incremental** path via `cloudReconcileDelta()` (index_v2.html:13885) —
  fetches only rows where `updated_at` advanced past the last-seen high-water mark;
  an idle station gets a near-empty response, not a full table dump.
- Full re-pulls (`loadFromCloud`) are still limited to infrequent triggers: tab
  focus/visibility change, realtime-socket reconnect, and a one-shot empty-board
  self-heal — the self-heal was itself patched (commit `821e680`) so it can't loop
  and force a full `select()` every poll forever.
- Other periodic timers (`checkEndOfDay`, `flushPendingWrites`, print-agent
  keepalive) do not perform per-tick cloud reads; `flushPendingWrites` early-exits
  when its queue is empty.

**Conclusion:** the fix from the July incident (commit `fc41367` + follow-ups) is
intact — no code-level regression as of this date. This is a point-in-time review,
not a guarantee against future changes; re-audit if a future daily egress reading
comes back unexpectedly high.
