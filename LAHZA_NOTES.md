# Lahza — Project Notes

**Last updated:** 2026-07-31

Lahza is a personal life management PWA built for a single user (nosirxongafforov@gmail.com). It covers daily habits, Islamic prayer tracking, learning, finances, health, weekly planning, and a historical calendar — all in one offline-capable mobile app, in Uzbek.

- **Live URL:** https://nosirxongafforov-web.github.io/Lahza
- **Repo:** https://github.com/nosirxongafforov-web/Lahza
- **Main file:** `index.html` (~2000 lines, single-file app — no build step, no framework)

---

## Tech Stack

| Layer | Detail |
|---|---|
| Hosting | GitHub Pages — auto-deploys on every push to `main` |
| Auth | Supabase Google OAuth (single allowed user) |
| Database | Supabase — `ht_store` table: `key TEXT, value JSONB` |
| PWA | `manifest.json` + `sw.js` (cache name: `lahza-v2`) |
| Prayer times | Aladhan API — Fergana, `lat=40.3834, lon=71.7849, method=1` |
| Icon | `icon.svg` — pure SVG crescent path, tilted 28°, `#E8E6E0` on `#0f0f0f` |
| JS libraries | `@supabase/supabase-js@2` via CDN (only external dependency) |

### Supabase Pattern

```js
dbGetSafe(key)   // reads ht_store, returns null on error, checks _cache first
dbSet(key, val)  // upserts ht_store, sets _cache, flips sync dot
_cache           // { [key]: value } — in-memory, session-scoped
```

All per-day data is keyed as `something_YYYY-MM-DD`. The `activeDate` global holds the currently viewed date.

---

## Design System

### Principles

- **Dark by default.** Light mode toggled via `body.light` class, persisted in `localStorage('lahza_theme')`.
- **Silver/white over gold.** Gold (`--gold`) appears only as accent borders and the active sidebar indicator. All text, buttons, and state indicators use the neutral `--text` / `--text2` / `--text3` scale.
- **Minimalist.** No decorative emoji in structural chrome. Data first.
- **Optimistic UI.** DOM updates instantly on user action; Supabase writes happen in the background.
- **Mobile-first, no bottom bar.** Navigation lives in a slide-in sidebar (260px) opened by the ☾ Lahza button in the top-left.
- **Single language: Uzbek.** All labels, toasts, and copy in Uzbek.

### CSS Variables

```css
/* Dark (default) */
:root {
  --bg:#111;  --bg2:#1a1a1a;  --bg3:#222;  --bg4:#2a2a2a;
  --border:rgba(255,255,255,0.07);  --border2:rgba(255,255,255,0.13);
  --text:#e4e2dc;  --text2:#8a8780;  --text3:#555250;
  --gold:#c9a84c;  --gold2:#e8c96a;  --gold-bg:rgba(201,168,76,0.10);
  --green:#4caf78;  --red:#e05c5c;  --blue:#5b9bd5;
  --purple:#9b7fd4;  --orange:#e07c3c;
  --r:12px;  --rs:8px;  --rx:6px;  --sw:260px;
}

/* Light (body.light) */
body.light {
  --bg:#f5f5f0;  --bg2:#ebebE6;  --bg3:#e0e0da;  --bg4:#d5d5cf;
  --text:#1a1a18;  --text2:#4a4a46;  --text3:#8a8a86;
  --border:rgba(0,0,0,0.08);  --border2:rgba(0,0,0,0.14);
}
```

Light mode has explicit overrides for every element group: cards, inputs, prayer grid, habit rows, schedule rows, calendar cells/popup, sidebar, top bar, user menu, toast, sleep quality buttons, goal/memo cards, edit rows, tables, stats, and the mini date picker.

### Theme Toggle

Lives in the user dropdown (avatar → click). Label shows "Kunduzgi rejim" (switch to light) or "Tungi rejim" (switch to dark). `toggleTheme()` flips `body.light`, saves to localStorage, and updates the label. `initTheme()` restores the preference on load.

---

## Navigation

### Top Bar
- **Left:** `☾ Lahza` button — opens sidebar
- **Center:** Date button (e.g. "Pa, 31 iyul") — clickable, opens mini date picker popup (see below)
- **Right:** sync dot · avatar/initials button → user dropdown

### Sidebar (260px slide-in)
Pages in order: Bugun · Namoz · Ilm · Moliya · Hisobot · Sog'liq · Reja · Tarix · Sozlamalar. Dark overlay closes it on tap. Active item has a left gold border.

### Date Navigation (Bugun / Namoz / Hisobot / Sog'liq pages)
`‹ date-label ›` — shift by ±1 day. Max future = tomorrow. Bugun page has no extra "Bugun" button (removed); other pages still have it.

### Mini Date Picker
Clicking the top-bar date button opens an inline monthly calendar popup anchored below it. Has `‹ ›` month navigation. Past days and today are selectable; days after tomorrow are greyed out. Picking a day sets `activeDate` and closes the picker. Closes on outside click. JS: `toggleDatePicker`, `dpShift`, `renderDatePicker`, `pickDate`. State: `dpMonth`, `dpYear`.

### `showPage(id, e)` flow
1. Hides all `.page` elements
2. Shows `#page-{id}`
3. Marks sidebar button active
4. Resets `activeDate` to today
5. Calls `closeSidebar()`
6. Calls `renderPage(id)`

---

## Pages

### Bugun (Today)
Main daily dashboard, keyed to `activeDate`.

**Date navigation:** ‹ `DD Mon` › — max = tomorrow. Banner "📅 Ertangi kun rejalari" appears when viewing tomorrow.

**Cards:**
| Card | Content |
|---|---|
| Metrics row | Umumiy %, Odatlar %, Namoz count — no inline color |
| Kunlik jadval | Schedule for the day. Uses `schedule_override_YYYY-MM-DD` if it exists, else falls back to `schedule`. Shows "✏ Bugunni/Ertani tahrirlash" button for today and tomorrow only. |
| Odatlar | Habit checklist. Optimistic toggle (instant DOM + background save). Disabled for future dates. |
| Namoz | Prayer status display (read-only on Bugun; edit on Namoz page) |
| Vazifalar | Daily tasks — add/check/delete |
| Kun eslatmasi | Free-text note for the day |

**Schedule override:** Tapping "✏ Tahrirlash" expands an inline editor inside the schedule card. "Saqlash" writes to `schedule_override_YYYY-MM-DD`; "Asosiyga qayt" deletes the override and restores the default schedule.

**Storage keys:** `hstate_YYYY-MM-DD`, `prayers_YYYY-MM-DD`, `daydata_YYYY-MM-DD`, `schedule`, `schedule_override_YYYY-MM-DD`

---

### Namoz (Prayer)
Tracks 5 daily prayers: Bomdod, Peshin, Asr, Shom, Xufton.

- Each prayer cycles through states: unset → `vaqtida` (on time) → `qazo` (late) → unset
- Prayer times fetched from Aladhan API, cached in `_cache['timings_YYYY-MM-DD']`
- Countdown to next prayer shown in a box at the top
- Asr time shown as plain dim text in schedule card (no emoji)
- Per-prayer notification toggle (🔔 / 🔕) stored in `notif_prefs`

**Storage:** `prayers_YYYY-MM-DD`, `pscores_YYYY-MM-DD`, `notif_prefs`

---

### Ilm (Learning)
Study and memorization tracker.

- **Qur'on memorization** — add sura, track daily portions, mark complete
- **Learning sessions** — log study time and topic
- **Goals** — titled goals with steps, deadline, and progress bar

**Storage:** `learn_sessions`, `memorizations`, `goals`

---

### Moliya (Finance)
Income and expense tracker.

- Add transactions (amount, category, description)
- Balance summary (income − expenses)
- Categorized transaction list

**Storage:** `transactions`

---

### Hisobot (Report)
Weekly self-review and reflection. Previously called "Muhasaba".

- Weekly mood trend — 7-bar CSS chart (no external library)
- Weekly sleep trend — 7-bar CSS chart
- Weekly review text input
- Date nav to browse past weeks

**Storage:** `review_YYYY-MM-DD`, `sleep_YYYY-MM-DD`, `mood_YYYY-MM-DD`

---

### Sog'liq (Health)
Daily health tracking.

- Sleep duration (hours input)
- Mood rating (1–5 scale with color)
- Water intake
- Exercise checkbox
- All fields stored inside `daydata_YYYY-MM-DD`

**Storage:** `daydata_YYYY-MM-DD` (nested: `sleep`, `mood`, `water`, `exercise`)

---

### Reja (Plan)
Goal-setting and weekly planning.

- **Maqsadlar** — long-term goals with title, description, deadline, step checklist
- **Haftalik reja** — free-text weekly plan
- **Ertangi reja** — tomorrow's plan (written the night before)

**Storage:** `goals`, `weekly_plan_YYYY-MM-DD`, `tomorrow_plan_YYYY-MM-DD`

---

### Tarix (History Calendar)
Performance calendar. New in this session.

**Month view:** Monday-first 7-column grid. Each past day shows a colored score dot:
- 🟢 ≥80% · 🟡 ≥60% · 🟠 ≥40% · 🔴 <40% · grey = no data

**Year view:** 4×3 month grid, each cell shows average monthly score with the same color scale.

**Day popup:** tapping any past day opens a modal with habit checklist, prayer status (all 5), task list, and free-text note for that day.

**Stats panel (bottom of page):**
- Faol kun (days with any data)
- O'rtacha ball (average score for the period)
- Eng uzun seria (longest streak of days ≥60%)

**Score formula:**
```
score = (habits_done / total_habits) * 40
      + (prayers_done / 5)           * 40
      + (tasks_done / total_tasks)   * 20
```

**JS:** `renderTarix`, `renderTarixMonth`, `renderTarixYear`, `renderTarixStats`, `tarixShift`, `setTarixView`, `getDayScore`, `showDayPopup`, `closeDayPopup`  
**State:** `tarixView` (`'month'`|`'year'`), `tarixMonth`, `tarixYear`  
**Storage:** reads from `hstate_*`, `prayers_*`, `daydata_*`, `habits_config`

---

### Sozlamalar (Settings)

**Kunlik jadval:**
- Edit schedule rows: time (HH:MM with auto-insert mask), label, tag
- Auto-sorts by time whenever a row is added
- Validates all rows before saving (rejects empty label or malformed time)
- "Saqlash" sorts + writes to `schedule`, refreshes view

**Odatlar:**
- Edit habit list: label and tag
- Tags: `deen` · `jism` · `aql` · `pul` · `ijt`
- "Saqlash" writes to `habits_config`, clears in-memory cache

**Bildirishnomalar:**
- Per-prayer notification toggles
- Stored in `notif_prefs` (delivery not yet wired — see pending)

**Hisob:**
- Shows user avatar and name
- Theme toggle (◐) — same as user dropdown button
- Sign out

**Storage:** `schedule`, `habits_config`, `notif_prefs`

---

## Auth

- Single allowed user: `nosirxongafforov@gmail.com`
- Google OAuth via Supabase, redirect back to `window.location.href`
- `onAuthStateChange` checks email on every session event; signs out any other account immediately with a toast
- Login screen (`#loginScreen`) overlays everything; hidden on successful auth

---

## PWA

```
manifest.json
  name: "Lahza"  short_name: "Lahza"
  display: standalone  theme_color: #c9a84c  lang: uz
  icons: [{ src: "icon.svg", sizes: "any", type: "image/svg+xml" }]

sw.js  (cache: lahza-v2)
  Precaches index.html, manifest.json, sw.js, icon.svg
  Serves from cache-first; falls back to network
```

**icon.svg** — pure SVG path, no emoji, no clip-path (avoids anti-alias artifacts). Waxing crescent drawn as outer arc + inner arc between the two circle intersection points `(70, 44)` and `(70, 148)`, rotated 28° clockwise around center `(96, 96)`.

---

## File Structure

```
Lahza/
├── index.html        # Entire app — HTML + CSS + JS, ~2000 lines
├── manifest.json     # PWA manifest (SVG icon, Uzbek lang)
├── sw.js             # Service worker (cache-first, lahza-v2)
├── icon.svg          # App icon — SVG crescent, 192×192 viewBox
└── LAHZA_NOTES.md    # This file
```

---

## Key JS Globals

### State variables

| Variable | Type | Purpose |
|---|---|---|
| `activeDate` | string `YYYY-MM-DD` | Currently viewed date across all pages |
| `schedData` | array \| null | In-memory schedule (null = not loaded yet) |
| `habitsData` | array \| null | In-memory habits (null = not loaded yet) |
| `_cache` | object | `{ key: value }` in-memory Supabase cache |
| `sb` | Supabase client | Initialized in `boot()` |
| `tarixView` | `'month'`\|`'year'` | Tarix calendar view mode |
| `tarixMonth` | 0–11 | Tarix current month |
| `tarixYear` | number | Tarix current year |
| `dpMonth` | 0–11 | Date picker current month |
| `dpYear` | number | Date picker current year |
| `_overrideOpen` | boolean | Whether schedule override editor is expanded |

### Core functions

| Function | Purpose |
|---|---|
| `boot()` | App entry — init Supabase, check auth, load Bugun |
| `renderPage(id)` | Dispatcher to per-page render functions |
| `showPage(id, e)` | Switch active page, reset date, close sidebar |
| `refreshPage()` | Re-render active page (used after date change) |
| `shiftDate(d)` | Move `activeDate` ±1 day, max = tomorrow |
| `goToday()` | Reset `activeDate` to today and refresh |
| `todayStr()` | Returns today as `YYYY-MM-DD` (local time) |
| `localDateStr(dt)` | `Date` → `YYYY-MM-DD` (local time) |
| `isToday(ds)` | Returns `ds === todayStr()` |
| `fmtF(ds)` | Format date as `"Du, 1 Yan"` for display |
| `updateDateLabels()` | Sync all `.dlbl` elements to `activeDate` |
| `dbGetSafe(key)` | Safe Supabase read, caches result |
| `dbSet(key, val)` | Supabase upsert, updates cache, sets sync dot |
| `getSched()` | Load schedule (cached in `schedData`) |
| `getHabits()` | Load habits (cached in `habitsData`) |
| `toast(msg)` | Show 2s bottom toast |
| `setSyncDot(state)` | Set sync dot to `ok`\|`saving`\|`err` |
| `toggleTheme()` | Flip `body.light`, save to localStorage, update label |
| `toggleSidebar()` / `openSidebar()` / `closeSidebar()` | Sidebar control |
| `toggleDatePicker(e)` | Open/close mini calendar popup in top bar |
| `dpShift(d)` | Navigate date picker month ±1 |
| `renderDatePicker()` | Render date picker grid |
| `pickDate(ds)` | Set `activeDate` from picker, close popup, refresh |
| `toggleHabit(id)` | Optimistic habit toggle |
| `toggleSchedOverride()` | Expand/collapse override editor on Bugun |
| `saveSchedOverride()` | Save per-day schedule to `schedule_override_YYYY-MM-DD` |
| `clearSchedOverride()` | Delete override, restore default schedule |
| `getDayScore(ds)` | Compute 0–100 score for any date |
| `scoreToColor(score)` | Map score to CSS color variable |
| `showDayPopup(ds, score)` | Open Tarix day detail popup |
| `closeDayPopup()` | Close day popup |
| `renderTarix()` | Render Tarix page (dispatches to month/year) |
| `tarixShift(d)` | Navigate Tarix month or year ±1 |
| `setTarixView(v)` | Switch between `'month'` and `'year'` |
| `sortSched()` | Sort `schedData` array by HH:MM |
| `fmtTimeInput(el, i)` | Auto-insert `:` in schedule time inputs |
| `saveSchedule()` | Validate + sort + save schedule to DB |

### Constants

```js
PRAYERS_ALL = ['Bomdod','Peshin','Asr','Shom','Xufton']
TAGS = ['deen','jism','aql','pul','ijt']
MN = ['Yanvar','Fevral',...,'Dekabr']
DU = ['Yakshanba','Dushanba',...,'Shanba']
```

---

## Storage Key Reference

| Key pattern | Page | Content |
|---|---|---|
| `schedule` | Sozlamalar / Bugun | Default daily schedule array |
| `schedule_override_YYYY-MM-DD` | Bugun | Per-day schedule override |
| `habits_config` | Sozlamalar / Bugun | Habit definitions array |
| `hstate_YYYY-MM-DD` | Bugun / Tarix | `{ [habitId]: boolean }` |
| `prayers_YYYY-MM-DD` | Namoz / Bugun / Tarix | `{ [prayerName]: 'vaqtida'\|'qazo' }` |
| `pscores_YYYY-MM-DD` | Namoz | Prayer quality scores |
| `daydata_YYYY-MM-DD` | Bugun / Sog'liq / Tarix | `{ tasks, note, sleep, mood, water, exercise }` |
| `review_YYYY-MM-DD` | Hisobot | Weekly review text |
| `sleep_YYYY-MM-DD` | Hisobot | Sleep chart data |
| `mood_YYYY-MM-DD` | Hisobot | Mood chart data |
| `learn_sessions` | Ilm | Study session log |
| `memorizations` | Ilm | Qur'on memorization progress |
| `goals` | Reja / Ilm | Goal objects with steps |
| `weekly_plan_YYYY-MM-DD` | Reja | Weekly plan text |
| `tomorrow_plan_YYYY-MM-DD` | Reja | Tomorrow plan text |
| `transactions` | Moliya | Transaction array |
| `notif_prefs` | Sozlamalar / Namoz | Per-prayer notification settings |

---

## Known Issues / Pending

| # | Issue | Notes |
|---|---|---|
| 1 | **Tarix performance** | `getDayScore` fires individual Supabase reads for each day in the calendar grid. Month view = up to 31 reads. Should batch or pre-cache. |
| 2 | **Notification delivery** | `notif_prefs` is stored per prayer but actual `setTimeout`/push reminders are not wired. Aladhan times are fetched; just need the scheduler. |
| 3 | **Offline write queue** | `dbSet` fails silently when offline. Needs a queue that retries when connectivity returns. |
| 4 | **Moliya — no charts** | No monthly breakdown or category chart. Currently a flat transaction list only. |
| 5 | **Ilm — basic UI** | Goals/sessions have no progress percentage visualization or completion graph. |
| 6 | **Hisobot — read-only charts** | Mood/sleep bars are rendered from stored data; no direct input on this page (input is on Sog'liq). |
| 7 | **Schedule override editor styling** | The override editor inline in Bugun is functional but doesn't fully match the Sozlamalar editor style. |
| 8 | **manifest theme_color** | `manifest.json` still has `theme_color: #c9a84c` (gold). Should be `#111111` to match the app chrome. |
