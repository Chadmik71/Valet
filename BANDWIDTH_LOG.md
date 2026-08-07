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

## 2026-08-08: found and fixed the actual leak (correction to the 08-07 audit above)

The 08-07 static-code audit above checked the *intended* code paths and looked clean —
but missed a live behavioral bug. Per-project dashboard reading for chatswood-valet
showed **7 Aug egress = 403.38 MB**, ~100% of it `PostgREST` (direct table reads, not
Realtime/Functions/Auth) — almost the full daily budget in one day.

Cross-checked against the Supabase API logs (`get_logs`, service `api`): a full
`entries` select (`select=*&order=time_in.desc&limit=2000`) plus full `app_settings`
select plus a `staff-auth` list call were firing **together, every ~8 seconds**,
continuously, all day.

Root cause: `reconcileFromCloud`'s "at most once per 8s" guard (index_v2.html:14239)
only throttles how *often* it runs — it doesn't distinguish reason. Only `poll`
(the 60s timer) took the cheap incremental-delta path; every other reason
(`reconnect`, `focus`, `visible`) always ran the expensive full `loadFromCloud()`
(entries + app_settings + staff). That's fine if reconnect/focus events are rare —
but if the realtime socket is flapping (bad wifi, laptop sleep/wake, a proxy
killing idle websockets), each `reconnect` event re-triggers a full reload, capped
only by the 8s floor — turning the "rare safety net" into a steady drip of full
table dumps, 24/7.

Fix (commit `43d8b3d`): full reconciles now have their own 60s cooldown
(`lastFullReconcileAt` / `FULL_RECONCILE_MIN_GAP_MS`), independent of the 8s poll
guard. Once a full reload has run, subsequent non-poll triggers fall back to the
cheap delta path until the cooldown clears — so a flapping socket degrades to
"delta every 8s" (near-zero egress) instead of "full table dump every 8s".

Underlying question not yet answered: *why* was the realtime socket reconnecting
that often on 7 Aug? Tenant-level Realtime logs looked stable (no tenant
restarts in the relevant window), so the flapping is more likely client-side
(network/device) than a Supabase-side issue. The 60s cooldown bounds the damage
either way, but if a future egress reading is still high, check whether the
realtime channel is genuinely unstable (dropped connections in the browser
console) rather than assuming this fix alone explains everything.

**Next check:** re-read chatswood-valet's per-project daily egress in a day or two
to confirm 8 Aug onward drops back to a small number.
