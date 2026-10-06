# Pip-Boy Dashboard

Single-file HTML dashboard (`index.html`) themed after Fallout 4's Pip-Boy interface.
Published with GitHub Pages so it can be opened from both phone and laptop. **Fast, easy access is priority #1.**

## Tech Stack
- Single self-contained HTML file (~3500 lines), no build tools. Runs from GitHub Pages (main use) or `file://` in Firefox
- Vanilla JS/CSS, no frameworks
- MapLibre GL JS for vector tile map (API: MapTiler, key stored in localStorage and in synced settings)
- All data persisted in localStorage under key `pipboy_metrics_v2`, optionally synced through a GitHub Gist

## Privacy rule (public repo)
- The repo is public. **Never put personal data in `index.html` or this file**: no addresses, coordinates of personal places, exam lists, names, tokens or keys
- Personal data lives only in the user's own data (localStorage + secret gist). `personlige-data.json` (git-ignored) holds the initial places/exams for a one-time import
- `.gitignore` is a whitelist: only `.gitignore`, `index.html`, `CLAUDE.md`, `README.md` are tracked. The folder also contains unrelated private files

## Architecture

### State Model
```js
{ sleep: [], screen: [], deadlines: [], tasks: [], goals: {}, places: [], groups: [], settings: {}, meta: {} }
// task: { id, text, desc, status, groupId, important, steps: [{id,text,done}], deadlineId, sub5min,
//         location, placeId, geo, date, time, gcalId, created }
// deadline: { id, title, date ('' = not known yet), time, priority, gcalId }
// place: { id, name, label, lat, lng, home? }   group: { id, name }
// settings: { googleClientId, maptilerKey }      meta: { updatedAt }  (sync: last write wins)
```
`normalizeState()` fills defaults (incl. `DEFAULT_GROUPS`) for loaded, imported and synced data.

### 6 Tabs
1. **Home** — daily summary, averages, goal of the day, urgent deadlines
2. **Calendar** — month grid / 30-day agenda of deadlines, dated tasks and Google Calendar events; day detail list
3. **Deadlines** — priority-based deadline tracker (urgent/high/normal/low); date may be empty ("not known yet", sorts last)
4. **Tasks** — sub-tabs:
   - *All Tasks*: form with one field per row (title, description, group, ★ important, deadline chips + quick-create, place, address, date, time, Easy-fix, subtasks). List is grouped: ★ Important first, then each group (collapsible), then ungrouped. Status flow: active → in_progress → finished/discarded. Default filter "Open"
   - *Easy-fix*: quick tasks under 5 minutes
   - *Reiseplan*: travel planner
5. **Map** — Oslo vector tile map with task markers (55 neighborhoods in `OSLO_LOCATIONS`) and the "Lagrede steder" (saved places) manager
6. **Log** — Sleep, Screen time and Data sub-sections. Data has export/import (import **merges** by id/date) and "Sync & Connections" (Gist sync, Google Client ID, MapTiler key)

### Header & display controls
- Top-left fixed controls (always visible): theme toggle, brightness (Dark/Med/Light), language (NO/EN)
- Header shows Oslo clock + localized date on the left; sync status badge on the right

### Language (NO / EN)
- Default Norwegian, saved as `pipboy_lang`. Table `I18N` holds `[english, norwegian]` pairs
- A MutationObserver translates every rendered text node / placeholder / title by exact match, both directions. Use `tr()` for strings built in JS
- Mark user content with `data-noi18n` (task titles, descriptions, steps, deadline titles, place/group names, `<option>`s with user data) so it is never translated
- When adding UI text: add the pair to `I18N`

### Theme System
Two visual themes, toggled by a button fixed in the top-left corner:
- **Fallout** (default) — Pip-Boy green-on-black aesthetic: `#14fe17` on `#0a0a0a`, Share Tech Mono font, CRT scanlines, screen glow, flicker animations
- **Standard** — clean, modern light look without CRT effects: warm white / bright grey background (Google Maps-like, e.g. `#f5f3ef`), dark text, green accent (`--std-accent`), system font

### Brightness Control
Three levels via `data-brightness="dark|medium|light"` on `<body>` (both themes): Dark = default (Standard: bright grey), Medium ~25% brighter, Light ~50% brighter (Standard: warm white).

### Sync (GitHub Gist)
- Token (fine-grained PAT, "Gists: read and write") stored per device in localStorage (`pipboy_gist_token`, `pipboy_gist_id`)
- Whole state saved as `pipboy-data.json` in a secret gist. `save()` stamps `meta.updatedAt` and pushes after 2 s; pull on start, on tab focus and every 3 min. Newer `updatedAt` wins
- `applyRemote()` replaces local state without triggering a push

### Google Calendar (two-way)
- Google Identity Services token flow (`initTokenClient`, scope `calendar.events`), no server. Needs an OAuth Web Client ID (`S.settings.googleClientId`) and an https origin (GitHub Pages); not available on `file://`
- Reads primary calendar events for the visible range; `gcalPush(kind,id)` creates/updates an event for a task or deadline and stores `gcalId`. Google events linked to a dashboard item are hidden to avoid duplicates

### Saved Places & Travel Planner
- The place with `home: true` is where every day starts and is always the first option in "Place / Area"
- Task location priority (`taskPoint(t)`): exact address `t.geo` > saved place `t.placeId` > Oslo area `t.location`
- Reiseplan chains a day: Home → task 1 → task 2 …, arriving by each task's time
- Routing via **Entur Journey Planner v3** (GraphQL, no key, header `ET-Client-Name`): transit (Ruter real-time), walk and bike; Ruter disruptions from leg `situations`. Google Maps only via deep links
- Rain warning: Open-Meteo hourly data. ⚠☔ when it rains at departure but was dry at the previous scheduled task (≤36 h back)
- Address search (`geocode()`): Entur geocoder + OpenStreetMap Nominatim, MapTiler as fallback

### Key Design Patterns
- XSS prevention via `escHtml()` helper
- Oslo timezone: `toLocaleTimeString('en-GB', { timeZone: 'Europe/Oslo' })`; `osloIso(date, time)` gives ISO with the Oslo offset
- `todayStr()` returns `YYYY-MM-DD` in Oslo time
- `uid()` generates unique IDs
- localStorage access for settings via `lsGet` / `lsSet` (try/catch)

### Map Specifics
- MapLibre GL JS with MapTiler vector tiles (OpenMapTiles schema); key prompt on first use (`pipboy_maptiler_key`, also synced in settings)
- Road color coding: red highways, orange car roads, blue bike paths, yellow pedestrian paths (dashed trails). Can be switched off ("Veifarger" → `ROAD_PLAIN`, `pipboy_map_roadcolors`)
- Map styles from `MAP_PALETTES` (`fallout` / `standard`), Auto follows theme (`pipboy_map_style`); brightness slider 40–160% (`pipboy_map_brightness`)
- Task markers with priority-based colors and popups; address/place markers are square; saved places shown as ★ (home ⌂)

## Style Rules
- All colors use CSS custom properties (`--pip-green`, `--pip-green-dim`, etc.)
- Buttons: class `pip-btn`, inputs: class `pip-input`; forms stack one field per row (`.input-row` is a column)
- Toast notifications via `toast(msg)` function (translated with `tr()`)
- Priority colors: urgent=red, high=amber, normal=green, low=dim-green
- Must work well at phone width (≥ 320px)

## Important Constraints
- Must remain a SINGLE file (no separate CSS/JS)
- Must work from GitHub Pages and from `file://` (Google Calendar only from https)
- External resources: CDN links (MapLibre, font), Google Identity Services script, and public APIs: MapTiler, Entur, Open-Meteo, OSM Nominatim, GitHub API (gist sync), Google Calendar API
- No cookies, no server-side anything
