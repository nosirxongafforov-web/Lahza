# Lahza — Project Notes

## What It Is

Lahza is a personal life management PWA built for a single user (nosirxongafforov@gmail.com). It covers daily habits, Islamic prayer tracking, learning, finances, health, weekly planning, and a historical calendar — all in one offline-capable mobile app, in Uzbek.

Live URL: https://nosirxongafforov-web.github.io/Lahza  
Repo: https://github.com/nosirxongafforov-web/Lahza  
Main file: `index.html` (single-file app, ~1700+ lines)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Hosting | GitHub Pages (auto-deploy on push to `main`) |
| Auth | Supabase Google OAuth |
| Database | Supabase (`ht_store` key-value table: `key TEXT, value JSONB`) |
| PWA | `manifest.json` + `sw.js` (cache name: `lahza-v2`) |
| Prayer times | Aladhan API — Fergana, lat=40.3834, lon=71.7849, method=1 |
| No build step | Plain HTML/CSS/JS, no framework |

### Supabase helpers
- `dbGetSafe(key)` — reads from `ht_store`, returns null on error
- `dbSet(key, value)` — upserts into `ht_store`
- `_cache` object — in-memory cache for repeated reads in the same session
- All per-day data keyed as `something_YYYY-MM-DD`

---

## Design Principles

- **Dark by default.** Background `#111`, cards `#1a1a2a` / `#222`. Light mode available via toggle.
- **Silver/white over gold.** Gold (`#c9a84c`) is used sparingly — accent borders and rare highlights only. Text and active states use `--text` / `--text2` / `--border2`.
- **Minimalist.** No decorative emoji in structural UI. Data over decoration.
- **Optimistic UI.** State changes update the DOM instantly; Supabase save happens in the background.
- **Mobile-first.** Designed for phone use. Sidebar nav replaces a bottom tab bar.
- **Single language: Uzbek.** All labels, toasts, and copy are in Uzbek.

### CSS Variables (dark default)
```
--bg:#111  --bg2:#1a1a1a  --bg3:#222  --bg4:#2a2a2a
--text:#e4e2dc  --text2:#8a8780  --text3:#555250
--gold:#c9a84c  --green:#4caf78  --red:#e05c5c
--blue:#5b9bd5  --purple:#9b7fd4  --orange:#e07c3c
--r:12px  --rs:8px  --rx:6px  --sw:260px (sidebar width)
```

Light mode overrides applied via `body.light` class; preference stored in `localStorage` as `lahza_theme`.

---

## Navigation

A slide-in sidebar (260px wide) opens when the user taps the ☾ Lahza logo in the top-left. Overlay behind it closes it on tap. Each nav item calls `showPage(id, event)`, which closes the sidebar and calls `renderPage(id)`.

---

## Sections

### Bugun (Today)
The main daily dashboard. Keyed to `activeDate` (YYYY-MM-DD). Users can navigate to yesterday or tomorrow (Reja banner appears for tomorrow).

**Cards:**
- **Metrics row** — Umumiy score, Odatlar %, Namoz count (no inline color, inherits theme)
- **Kunlik jadval** — Scheduled activities for the day. Falls back to default `schedule` key; if `schedule_override_YYYY-MM-DD` exists in DB, that is used instead. Has "✏ Bugunni tahrirlash" button (today only) to create a per-day override.
- **Odatlar** — Habit checklist. Optimistic toggle (instant DOM update, background save). Disabled for future dates.
- **Namoz** — Prayer tracking (read from `prayers_YYYY-MM-DD`)
- **Vazifalar** — Daily tasks
- **Kun eslatmasi** — Free-text note

**Storage keys used:**
`hstate_YYYY-MM-DD`, `prayers_YYYY-MM-DD`, `daydata_YYYY-MM-DD`, `schedule`, `schedule_override_YYYY-MM-DD`

---

### Namoz (Prayer)
Tracks the 5 daily prayers: Bomdod, Peshin, Asr, Shom, Xufton.  
Status options: `vaqtida` (on time), `qazo` (late), or unset.  
Prayer times fetched from Aladhan API and cached in `_cache`.  
Shows countdown to next prayer. Asr time shown as plain dim text (no emoji).

**Storage:** `prayers_YYYY-MM-DD`, `pscores_YYYY-MM-DD`

---

### Ilm (Learning)
Study and memorization tracker. Includes:
- Qur'on memorization progress
- Learning sessions log
- Goals list

**Storage:** `learn_sessions`, `memorizations`, `goals`

---

### Moliya (Finance)
Income and expense tracker.  
Categorized transactions, balance summary.

**Storage:** `transactions`

---

### Hisobot (Report)
Previously called "Muhasaba". Weekly review and reflection page.  
Mood and sleep bar charts (CSS-only, no external chart library).

**Storage:** `review_YYYY-MM-DD`, `sleep_YYYY-MM-DD`, `mood_YYYY-MM-DD`

---

### Sog'liq (Health)
Daily health tracking — sleep duration, mood rating, water intake, exercise.

**Storage:** `daydata_YYYY-MM-DD` (nested fields)

---

### Reja (Plan)
Weekly planning grid (7-column layout).  
Also shows tomorrow's plan as a dedicated card.

**Storage:** `weekly_plan_YYYY-MM-DD`, `tomorrow_plan_YYYY-MM-DD`

---

### Tarix (History Calendar)
Monthly and yearly calendar views showing daily performance scores.

**Month view:** 7-column grid (Monday-first). Each day shows a colored dot based on score (green ≥80%, yellow ≥60%, orange ≥40%, red below). Tapping a past day opens a popup with full habit/prayer/task breakdown.

**Year view:** 4×3 grid of months, each showing average monthly score with color coding.

**Stats panel:** Active days count, average score, longest streak (≥60% days).

**Score formula:** `habits * 40% + prayers * 40% + tasks * 20%`

**Storage:** reads from `hstate_*`, `prayers_*`, `daydata_*`, `habits_config`

---

### Sozlamalar (Settings)
- **Kunlik jadval** — Edit schedule rows (time HH:MM with input mask, label, tag). Auto-sorts by time on add. Validates before save.
- **Odatlar** — Edit habit list (label, tag). Tags: `deen`, `jism`, `aql`, `pul`, `ijt`.
- **Bildirishnomalar** — Notification preferences (per-prayer toggles).
- **Hisob** — User info, theme toggle (moves to user dropdown), sign out.

**Storage:** `schedule`, `habits_config`, `notif_prefs`

---

## Auth

Single allowed user: `nosirxongafforov@gmail.com`.  
If a different Google account signs in, they are signed out immediately with a toast.  
Auth state managed by `sb.auth.onAuthStateChange`.

---

## PWA

- `manifest.json` — app name, icons, `display: standalone`, `theme_color: #111111`
- `sw.js` — precaches key assets (cache name `lahza-v2`), serves from cache first
- Works offline after first load

---

## Pending / Known Issues

- [ ] **Tarix popup performance** — `getDayScore` fires individual Supabase reads per day in month/year views; consider batching or caching results
- [ ] **Schedule override UI polish** — override editor in Bugun is functional but could match Sozlamalar styling more closely
- [ ] **Notification delivery** — `notif_prefs` is stored but actual `setTimeout`/`setInterval` prayer reminders not yet wired
- [ ] **Hisobot charts** — CSS bar charts are read-only; no input for sleep/mood from this page (done via Sog'liq)
- [ ] **Ilm page** — goals and sessions UI is basic; no completion percentage or progress visualization
- [ ] **Moliya page** — no monthly summary or category breakdown chart yet
- [ ] **Offline write queue** — `dbSet` fails silently when offline; no retry queue

---

## File Structure

```
Lahza/
├── index.html        # Entire app (HTML + CSS + JS, single file)
├── manifest.json     # PWA manifest
├── sw.js             # Service worker
├── icon-192.png      # PWA icon
├── icon-512.png      # PWA icon
└── LAHZA_NOTES.md    # This file
```

---

## Key JS Globals

| Variable | Purpose |
|---|---|
| `activeDate` | Currently viewed date (YYYY-MM-DD) |
| `schedData` | In-memory schedule array (null = not yet loaded) |
| `habitsData` | In-memory habits array (null = not yet loaded) |
| `_cache` | Key→value in-memory cache for DB reads |
| `sb` | Supabase client instance |
| `tarixView` | `'month'` or `'year'` for Tarix page |
| `tarixMonth` / `tarixYear` | Currently viewed period in Tarix |

| Function | Purpose |
|---|---|
| `renderPage(id)` | Dispatcher — calls the right render function |
| `showPage(id, e)` | Switches active page + closes sidebar |
| `shiftDate(d)` | Navigates activeDate ±1, max = tomorrow |
| `todayStr()` | Returns today as YYYY-MM-DD |
| `localDateStr(dt)` | Converts Date object to YYYY-MM-DD local |
| `dbGetSafe(key)` | Safe Supabase read with cache |
| `dbSet(key, val)` | Supabase upsert |
| `toast(msg)` | Shows temporary bottom toast |
| `toggleTheme()` | Toggles light/dark, updates label |
| `getDayScore(dateStr)` | Computes 0-100 score for any date |
| `showDayPopup(date, score)` | Opens Tarix day detail popup |
