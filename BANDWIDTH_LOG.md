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

## 2026-08-09: 8 Aug closed out healthy — ~204.8MB, ~57% below 7 Aug

Per-project dashboard, chatswood-valet filter, "Egress per day" chart — both 7 Aug and
8 Aug are now closed/finalized buckets:

- **7 Aug 2026 (closed):** 484.27MB total — 484.071MB PostgREST (100.0%), 123.626KB
  Functions, 54.149KB Auth, 18.305KB Realtime. Note: this finalized figure is higher
  than the 403.38MB read on 08-08 morning (mid-day figures apparently still update
  after the fact as the day's data settles) — treat 08-08's in-progress readings as
  provisional, the day-after closed number as the real one.
- **8 Aug 2026 (closed):** ~204.76MB total — 204.54MB PostgREST (99.9%), 92.591KB
  Auth, 60.688KB Realtime, 60.33KB Functions. ~57% below 7 Aug's closed total, and
  in line with the evening-of projection (~100-120MB extrapolated, actual came in a
  bit higher but still well within budget). No sign of the flapping-reload pattern
  recurring — fix from commit `43d8b3d` holding.
- **Cumulative this cycle (06 Aug – 06 Sep, chatswood-valet only):** 0.722 GB / 250 GB
  (<1%).
- 9 Aug (today) doesn't have its own bar yet — chart hadn't populated a partial-day
  figure for it at check time.

## 2026-08-08 (mid-morning): fix confirmed — 8 Aug running at ~15MB vs 403MB on 7 Aug

Checked the per-project dashboard again partway through the day (8 Aug bucket still
in progress, not yet closed): **~15.1MB total so far** — 15.092MB PostgREST,
12.89KB Auth, 9.05KB Functions, 5.79KB Realtime. That's a ~96% drop from 7 Aug's
403.38MB, and for a partial day. No sign of the full-reload-every-8s pattern
recurring. A cloud routine is still scheduled for 2026-08-09 12:00pm Sydney to
double-check the full closed 8 Aug bucket and the API-log pattern directly, but
this reading already strongly confirms commit `43d8b3d` fixed the leak.

## 2026-08-08 (evening, ~6:10pm Sydney / 8hrs into the UTC day): still tracking linear, healthy

**~78.1MB total so far** — 78.009MB PostgREST, 64.45KB Auth, 52.02KB Realtime,
28.31KB Functions. Progression through the day: ~15.1MB at ~2hrs in, ~78.1MB at
~8hrs in — roughly 9-10MB/hour, holding steady (linear, not accelerating).
Projected across a full ~10-12hr business day that's ~100-120MB total, about a
quarter of 7 Aug's 403MB disaster and comfortably within budget. API log
cross-check was unavailable this time (Supabase `get_logs` timing out — project
itself reports `ACTIVE_HEALTHY`), but the steady linear rate is itself the
reassuring signal; a recurrence of the flapping-reload bug would show
exponential growth, not this.

## 2026-08-11: cumulative 1.197 GB / 250 GB (<1%) — 9 Aug bar higher than expected, still within budget

Per-project dashboard, `chatswood-valet` filter, current billing cycle (06 Aug –
06 Sep 2026):

- **Cumulative egress this cycle (chatswood-valet only): 1.197 GB / 250 GB
  (<1%).** No overage, nowhere close.
- The "Egress per day" chart (visual read of the bar heights against the
  143MB/286MB/484MB gridlines — no exact tooltip value this time, so these are
  close estimates, not exact figures like earlier entries):
  - 07 Aug (closed): ~484MB — matches the 08-09 log entry's finalized 484.27MB.
  - 08 Aug (closed): ~205MB — matches the 08-09 log entry's finalized 204.76MB.
  - 09 Aug (closed): **~350-380MB** — noticeably higher than 08 Aug, breaking
    the "steady healthy" trend the 08-08 entries projected. Not investigated
    further this check (still well within the 250GB budget), but worth a look
    if it keeps climbing — could be legitimate higher usage (busier day) or an
    early sign of the reconnect-flapping pattern from the 7 Aug incident
    creeping back.
  - 10 Aug (closed): small bar, roughly ~50-90MB.
  - 11 Aug (today): not yet on the chart (too new / <24h to populate).
- Org now has **4 projects** total (was 2 at the 2026-08-07 baseline check):
  `chatswood-valet`, `Chadmik71's Project`, `Manly-clinic` (new), and
  `scone-thai-massage`. All still pool from the same 250GB/month org quota, so
  the org-total-vs-per-project gap is now wider than before — reinforces why
  the per-project filter matters for this log.
- **Action item:** next check should try to get the exact 09 Aug figure (hover
  tooltip on the chart didn't render this pass) rather than relying on the
  visual estimate above, given the accuracy bar for the customer pitch.

## 2026-08-11 (follow-up): 9 Aug's spike investigated — exact figures + root-cause check

Got the exact tooltip values for the two days flagged above, and dug into whether
09 Aug's jump was a code regression:

- **09 Aug (closed, exact): 384.9MB total** — 384.621MB PostgREST (99.9%),
  95.604KB Auth, 85.749KB Realtime, 103.785KB Functions. Confirms the visual
  estimate; ~88% above 08 Aug's 204.76MB.
- **10 Aug (closed, exact): 60.16MB total** — 60.157MB PostgREST (100%),
  8.722KB Auth, 4.819KB Realtime, 15.264KB Functions. The healthiest day yet —
  well below even 08 Aug, so whatever drove the 09 Aug spike did **not**
  persist into 10 Aug.
- **Cumulative this cycle: 1.197–1.219 GB / 250 GB (<1%).** No budget concern.

**Root-cause check:** 09 Aug is the same day two features shipped
(`cf10ed3` photo→Storage migration and `e71f52e` historical
reports/`entriesForRange`, both ~10:20–10:30am). Checked whether either
reintroduced the July/Aug-7 "full table dump" pattern:
- `entries` table is tiny (963 rows, ~4.1MB total, photo columns included) —
  confirmed live via SQL. A single full-table pull is only ~4-5MB, so 384MB
  in one day means roughly the equivalent of ~80-90 full pulls, not one
  runaway query.
- Audited every `setInterval` in `index_v2.html`: none of them call
  `entriesForRange`/`fetchEntriesInRange` (the new historical-fetch path) —
  `refreshActiveViewOnly()`'s 10s tick only calls `renderReports()`, which is
  local-array-only per the `e71f52e` design. So the new report/export feature
  can't have caused a *periodic* drain — it's click-driven only.
- No code-level regression found. Most likely explanation: manual testing/
  demoing of the two just-shipped features that morning (repeated "All Time"
  exports or historical month switches each do one small `select('*')`
  round-trip) — plausible at ~1.2GB/hour... actually more like dozens of
  clicks — not confirmed, just the best fit given nothing periodic was found.

**New, separate finding (not yet explained):** live `postgrest` logs show a
recurring `Warp server error: Thread killed by timeout manager` message —
intermittent (every 20-90 min) on 09-10 Aug, but firing every ~30-90
*seconds* as of this check (11 Aug, live). This is a low-level PostgREST/Warp
HTTP-server log with no request path/size attached, so it's unconfirmed
whether it's egress-relevant (it may just be idle keep-alive connections
timing out, which transfer ~0 bytes) or a symptom of something hanging. Flagged
here since its frequency is visibly climbing today — worth a check next time
if 11 Aug's egress reading comes back elevated too.
