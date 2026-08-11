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

## What to do

1. Do the actual requested work first (the code/investigation/deliverable).
2. Append **one new row** to the table in `FIX_LOG.md`, matching the existing
   columns exactly:

   | Date | Asked | Issue found | Result / fix | Status | Ref |
   |---|---|---|---|---|---|

   - **Date** — today, Sydney-local (`YYYY-MM-DD`)
   - **Asked** — what the user asked for, in their own words/paraphrased
   - **Issue found** — the root cause if it was a bug; `N/A — ...` if not
   - **Result / fix** — what was actually changed or built, concrete enough
     that a future read tells you whether this is already solved
   - **Status** — `✅ Done`, `⚠️ Partial` / needs follow-up, etc.
   - **Ref** — file(s), commit hash, or artifact name

   Don't edit old rows except to update Status if something regresses —
   append a new row instead, per the file's own stated convention.
3. Commit the `FIX_LOG.md` change (bundle it with the code commit if there is
   one; a standalone commit if it's a log-only entry).
4. Push to **both** remotes per the project's push convention — see
   `CLAUDE.md`: `git push origin main; git push neworigin main`.
