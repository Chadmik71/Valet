# Valet App — Claude Instructions

## ⚠️ ALWAYS PUSH TO BOTH SITES

Every commit MUST be pushed to BOTH GitHub Pages repos, every time — never just one:

```
git push origin main; git push neworigin main
```

- `origin` → `Chadmik71/Valet` → https://chadmik71.github.io/Valet/ (old/fallback)
- `neworigin` → `ChatswoodChaseValet/ChatswoodChaseValet.github.io` → https://chatswoodchasevalet.github.io/ (**canonical — this is the one staff use**)

The two repos are completely separate. Pushing only `origin` leaves the branded site staff actually use running old code, silently. This has bitten us more than once (2026-07-04, 2026-07-28). After pushing, changes go live in ~30–60s.

## Project basics

- Single-file app: `index_v2.html` is the served dashboard (not `index.html`).
- Backend: Supabase (`https://ebqiitxiyzzbkgyfypss.supabase.co`), tables `entries`, `staff`, `app_settings`, `audit_log`. Anon key hardcoded in `index_v2.html`.
- `entries` columns are snake_case; JS uses camelCase — `jsToDb()` / `dbToJs()` convert. Update BOTH when adding a column.
- Timestamps stored UTC; always display Sydney time (`AT TIME ZONE 'Australia/Sydney'` in SQL).
- During testing: commit + push directly to main without asking for confirmation.
- Keep `.nojekyll` at repo root (Pages drops binary assets without it).
