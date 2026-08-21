# SOC Roster

The full roster **generator** app (`soc_roster_final_v27.html`, unmodified —
carryover, leave, NS Teams, the whole generation engine, local styled
Excel export) hosted on GitHub Pages, with one addition: a working
**"Push to Google Sheets"** button that writes the generated roster
straight into a Sheet via a Cloudflare Worker.

This is a private, admin-only tool. Only accounts flagged `Yes` in
`Employee_Master` column F can push, list months, or edit via the Worker —
see "How access is actually controlled" below.

---

## What's in this repo

- `index.html` — the original generator app, with `WORKER_URL` and
  `GOOGLE_CLIENT_ID` filled in and `pushToSheets()`'s admin check fixed to
  actually verify admin status (previously it accepted any signed-in user).
- `robots.txt` — keeps the Pages site out of search indexing.

## What's deployed separately (not in this repo)

- `worker.js` — the Cloudflare Worker, pasted into the Cloudflare
  dashboard's code editor directly (no `wrangler`, no build step). Kept
  out of this repo since it's deployed by copy-paste, not by CI.

---

## Architecture

```
GitHub Pages (this repo, private)
  index.html  — the full generator: build a roster locally, then either
                download it as a styled .xlsx (unchanged, local, always
                works) or push it straight to Google Sheets
        │
        │  fetch(WORKER_URL, { action:'publishRoster', token, ...rosterData })
        ▼
Cloudflare Worker (soc-roster-lite)
  verifies the caller's Google identity → checks Employee_Master for the
  admin flag → creates the month's two Sheets tabs if missing → writes
  the full grid + a formula-driven summary tab
        │
        ▼
Google Sheet (1T_Z8RqAdP-jD4W_sZJAf9TBeicoTcsf-caOk2L61UdI)
  Employee_Master              — admin allowlist + name/dept/type
  "<Month> <Year>"              — main roster grid (e.g. "September 2026")
  "<Month> <Year> Summary"      — allowance summary, formulas cross-
                                   reference the main tab, recalculates
                                   automatically in Sheets
```

The browser never talks to the Sheets API directly. Every read/write goes
through the Worker, the only thing holding Sheets-capable credentials (a
service account key).

---

## The three Worker actions actually used by this app

| Action | Used by | What it does |
|---|---|---|
| `publishRoster` | "Push to Google Sheets" button | Creates the month's two tabs if they don't exist, clears stale content, writes the full grid + summary with live `COUNTIF`/cross-sheet formulas |
| `listMonthTabs` | Admin badge check on sign-in | Confirms the signed-in account is genuinely an admin (fails closed if not) |
| `editShift`, `getRosterMonth` | Not called from this page | Kept in the Worker for a future lightweight viewer/editor if one gets built later; harmless to leave in |

**Known limitation, on purpose:** `publishRoster` writes values and
formulas only — no cell colors, merges, or freeze panes in the live
Google Sheet. The fully styled, colored version is still what
`doExport()` produces locally (unchanged). Building matching visual
formatting server-side (Sheets `batchUpdate` cell-format requests) is a
real follow-up if the plain Sheet view isn't good enough day to day — flag
it and it can be added.

---

## Google Sheet tab layout the Worker assumes

**`Employee_Master`** (data from row 3):

| Col | Meaning |
|---|---|
| A | Sl No |
| B | Employee Name |
| C | Department |
| D | Email |
| E | Type (`Rotating` / `General`) |
| F | Admin flag — must be exactly `Yes` |

**`<Month> <Year>`** (e.g. `September 2026`) — written by `publishRoster`,
matches the app's own local export layout: row 5 day-of-week header, row 6
day numbers from column C, row 7+ employee rows (NSOC employees suffixed
`- NSOC`), then `Total MS/GS/AS/NS` rows, then the code legend.

**`<Month> <Year> Summary`** — paired tab, written by `publishRoster`,
formula-driven, cross-references the main tab by name. If you ever rename
a month tab manually in the Sheet, its Summary formulas will break — don't
rename tabs by hand, re-push instead.

---

## How access is actually controlled

Two independent server-side gates, both enforced inside the Worker — the
URL being unlisted is a courtesy, not the security boundary:

1. **Google OAuth consent screen (Testing status, Test Users list)** — only
   specific Google accounts can complete sign-in at all.
2. **`Employee_Master` column F** — even a valid signed-in token is
   rejected by every Worker action unless that email has `Yes` in column
   F. This is the check that matters day to day.

To add or remove an admin, update both: the OAuth test users list, and the
`Yes` flag in `Employee_Master`.

---

## Deployment record

### Google Cloud
- Project: `roster-generator-506119`
- Sheets API enabled
- OAuth consent screen: External, Testing, test users = admin emails only.
  Scopes: `userinfo.email`, `userinfo.profile` — no Sheets scope requested
  client-side, ever.
- OAuth Client ID (Web), Authorized JavaScript origin:
  `https://neel012345.github.io`
- Service account: `soc-roster-lite-sa@roster-generator-506119.iam.gserviceaccount.com`
  — **rotate the key immediately if it is ever exposed outside Cloudflare's
  secret storage** (chat, commit history, screenshots, etc.)

### Google Sheet
- Shared with the admin's Google account: **Editor**
- Shared with the service account email: **Editor**

### Cloudflare Worker
- Name: `soc-roster-lite`
- Code: `worker.js`, pasted via the Cloudflare dashboard
- Secrets: `SA_EMAIL`, `SA_PRIVATE_KEY`
- URL: `https://soc-roster-lite.navoneel-itorizin-87e.workers.dev`

### This repo / GitHub Pages
- Private repo
- `index.html` has `WORKER_URL`/`GOOGLE_CLIENT_ID` filled in directly —
  safe to commit, since a Client ID isn't a secret and no Sheets
  credentials live client-side
- Settings → Pages → deploy from this branch, root
- `robots.txt` present at repo root

---

## Updating

- **Worker logic changes** → edit `worker.js`, paste into the Cloudflare
  dashboard, Deploy.
- **App changes** → edit `index.html`, commit/push, Pages redeploys.
- **New admin** → add their email as an OAuth test user, add a `Yes` row
  in `Employee_Master`.
- **Revoke an admin** → remove the `Yes` flag (immediate) and remove them
  from OAuth test users.
