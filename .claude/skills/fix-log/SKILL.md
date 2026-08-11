---
name: fix-log
description: Log every fix, feature add, or investigation request to FIX_LOG.md at the repo root. Use whenever the user asks to fix a bug, add or change a feature, or investigate an issue in the Valet app.
---

This repo keeps a running record of every fix/add/investigate request in
`FIX_LOG.md` at the repo root, so it's easy to check "have I already asked
for this?" or "was this fixed already?" without digging through chat history.

## When to use this

Any time the user asks to:
- fix a bug
- add or change a feature
- investigate something that isn't obviously a bug (e.g. "why is X happening")

Deliverable-only requests (PDFs, summaries, docs) also get a row — use
"N/A — deliverable request" as the Issue found.

**Also:** at the start of a session (or when it's otherwise relevant), skim
the "🔭 Open follow-ups" section at the top of `FIX_LOG.md`. If the current
conversation touches one of those items (e.g. the user asks to check
bandwidth and there's an open egress follow-up), surface it and, if it's now
resolved, close it out per step 4 below instead of leaving it stale.

## What to do

1. Do the actual requested work first (the code/investigation/deliverable).
2. Append **one new row** to the table in `FIX_LOG.md`, matching the existing
   columns exactly:

   | Date | Asked | Issue found | Result / fix | Status | Follow-up | Ref |
   |---|---|---|---|---|---|---|

   - **Date** — today, Sydney-local (`YYYY-MM-DD`)
   - **Asked** — what the user asked for, in their own words/paraphrased
   - **Issue found** — the root cause if it was a bug; `N/A — ...` if not
   - **Result / fix** — what was actually changed or built, concrete enough
     that a future read tells you whether this is already solved
   - **Status** — `✅ Done`, `⚠️ Partial` / needs follow-up, etc.
   - **Follow-up** — `—` if the fix is confirmed/self-evident (e.g. a
     deterministic code change you can verify by reading the diff). Otherwise
     a short note on what still needs confirming and how: e.g. "⏳ Watching —
     confirm next day's egress reading actually drops". Use this for anything
     whose real-world effect can't be proven in the same session — timing
     races, "should reduce X" claims that need a live metric, fixes for
     intermittent symptoms, etc.
   - **Ref** — file(s), commit hash, or artifact name

   Don't edit old rows except to update Status/Follow-up if something
   changes — append a new row instead, per the file's own stated convention.
3. If the new row has a non-`—` Follow-up, add a matching bullet to the "🔭
   Open follow-ups" section at the top of the file.
4. When a previously-open follow-up gets confirmed (positively or
   negatively) in a later session, update that row's Follow-up cell in place
   (e.g. "✅ confirmed 2026-08-13 — egress dropped to 8MB/day") and remove
   its bullet from "🔭 Open follow-ups".
5. Commit the `FIX_LOG.md` change (bundle it with the code commit if there is
   one; a standalone commit if it's a log-only entry).
6. Push to **both** remotes per the project's push convention — see
   `CLAUDE.md`: `git push origin main; git push neworigin main`.
