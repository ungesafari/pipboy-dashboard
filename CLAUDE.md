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
{ sleep: [], screen: [], deadlines: [], tasks: [], goals: {}, places: [], groups: [], courses: [], settings: {}, meta: {} }
// task: { id, text, desc, status, groupId, important, steps: [{id,text,done}], deadlineId, sub5min,
//         location, placeId, geo, date, time, gcalId, created }
// deadline ("important date"): { id, title, date ('' = not known yet), estDate (assumed EARLIEST date when unknown), time, priority, gcalId }
// course: { id, name, short, code, examId, start, end, placeId, calMatch, weeks: [{ id, week, from, to, note,
//           lessons: [{ id, label, date, start?, end?, topics: [], items: { class, hw, e1, e2: [strings] } }] }],
//           lectures: [{ id, title, platform, url, chapter, minutes, order }], done: { itemId: date }, watched: { lectureId: date } }
// place: { id, name, label, lat, lng, home? }   group: { id, name }
// settings: { googleClientId, maptilerKey }      meta: { updatedAt }  (sync: last write wins)
```
`normalizeState()` fills defaults (incl. `DEFAULT_GROUPS`) for loaded, imported and synced data.

### 7 Tabs
1. **Home** — daily summary, averages, goal of the day, upcoming important dates
2. **Calendar** — views 1 day / **3 days (default)** / 7 days as hour-based timetable grids, plus Month and Agenda. Shows important dates, dated tasks, course lessons and Google Calendar events; day detail list. The **"Important dates"** button opens the former Deadlines tab as a panel here (form + list)
3. **Tasks** — sub-tabs:
   - *All Tasks*: form with one field per row (title, description, group, ★ important, deadline chips + quick-create, place, address, date, time, Easy-fix, subtasks). List is grouped: ★ Important first, then each group (collapsible), then ungrouped. Status flow: active → in_progress → finished/discarded. Default filter "Open"
   - *Easy-fix*: quick tasks under 5 minutes
   - *Reiseplan*: travel planner
4. **Courses** ("Fag") — digitized progress plans: weeks → lessons → checkable items per level (I timen / Lekser / E1 / E2), progress per level, "behind" count (in-class + homework from past lessons), lecture tracking (Itslearning / FlipClass videos), per-course settings and JSON import
5. **Body** ("Kropp") — interactive 3D model of the male body (muscles, tendons, ligaments): rotate 360°, tap to zoom in, names in Latin / Norwegian / English
6. **Map** — Oslo vector tile map with task markers (55 neighborhoods in `OSLO_LOCATIONS`) and the "Lagrede steder" (saved places) manager
7. **Log** — Sleep, Screen time and Data sub-sections. Data has export/import (import **merges** by id/date) and "Sync & Connections" (Gist sync, Google Client ID, MapTiler key)

### Important dates (formerly "Deadlines")
- A date may be unknown: then `estDate` is the assumed **earliest** possible date. `effDate(d)` = date || estDate is used for sorting, countdowns ("≤ N days, at the earliest") and the calendar (dashed/italic). This avoids false comfort for unknown exam dates
- Only real dates can be pushed to Google Calendar

### Courses
- Plans are data, never code (they contain personal/teacher info). Personal plan file: `fag-r1-fysikk.json` (git-ignored), imported via Log → Data
- Item strings expand: `"1.14-1.17"` → one checkbox per exercise; `*` = programming task. Item id = `lessonId|level|label`, so progress (`done`) survives re-importing a corrected plan
- Lessons appear in the calendar with the course's default time (`start`/`end`, per-lesson override). If a Google event overlaps and its title contains `calMatch`, the lesson is hidden in the grid to avoid duplicates
- Lectures import: paste JSON `{ "lectures": [{ id, title, platform, url, chapter, minutes, order }] }` in the course settings; merged by id, `watched` progress kept

### Body (3D anatomy)
- Three.js r128 + OrbitControls from jsdelivr, loaded only when the tab opens (`initBody`); render loop runs only while the tab is visible
- Data-driven: `BODY_REGIONS` (areas), `BODY_GROUPS` (muscle groups), `BODY_PARTS` ({ id, type muscle|tendon|ligament, region, group, la, no, en, shapes, side }). Shapes are built for the LEFT side (x > 0) and mirrored; `side: 'mid'` for midline structures
- Shapes: `seg` (spindle/rod via LatheGeometry between two points) and `ell` (ellipsoid). The neutral base body (`bodyBaseShapes`) is translucent and not clickable
- Selecting fades everything else (so deep structures like the cruciate ligaments are visible) and flies the camera to the structure from outside its nearest limb/torso axis (`bodyAxes`)
- Planned next (not built yet): function text per group and the 3 most common complaints per region (for a fit, non-overweight person who sits at a desk), with causes and treatment — attach to `BODY_REGIONS` / `BODY_GROUPS`

### Map facts (clickable areas)
- Chips "Tap on: Country / County / Municipality / City district" (`geoMode`) set the level of detail. Levels follow `GEO_LEVEL_MODE`, and zooming by hand changes the mode. A click takes the finest hit at or above the mode, so outside the four cities a district click gives the municipality, and outside Norway it gives the country
- Borders are TopoJSON from jsDelivr, loaded on demand (`geoLoad`): world-atlas for countries, AnalyseABO/Kart-fylker-og-kommuner-json (pinned to a commit) for Norway. Custom layers are added in `geoAddLayers`; `setStyle` must use `{ diff: false }` or `style.load` does not fire and they vanish
- Districts exist for Oslo, Bergen, Stavanger and Trondheim. SSB uses `030105a`-style codes for Oslo (`ssbCode`)
- Figures, fetched per click and cached in `GEO.cache`. **Every figure shows its own year**:
  - Countries: Data360 (`wbLatest`) for population, GDP, exports, imports and sector shares. **Do not use `api.worldbank.org/v2/.../indicator` from the browser**: most of its servers omit the CORS header. `/v2/country/ISO` is fine. Avoid its JSONP, it would run remote script next to the stored tokens
  - Top exports: Harvard Atlas GraphQL (HS92 chapters + services, Norwegian names in `PRODUCT_NO`); Norway uses SSB 08809 (goods, SITC)
  - Income sources: three World Bank sectors; Norway gets SSB 09170 (value added by industry)
  - County/municipality/district: SSB 10987 population, 06944 median income, 07984 jobs by industry, 13540 municipal income, 09434 education
- UN Comtrade has no CORS, so it can't be called from the page

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
- Google events can be edited, created and deleted from the calendar (`openGev` / `saveGev` / `deleteGev`, panel `#gevPanel`). Multi-day events: only title/place/description
- Two-way for linked items: `gcalSig` stores title|date|time as last synced. On every fetch `gcalPullLinked()` pulls Google-side edits into items not changed locally; the **"⇅ Sync Google"** button (`gcalSyncAll`) pushes local edits. Pushing keeps the existing event's duration

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
- Level buttons Oslo / Norway / Scandinavia / Europe / World (`MAP_LEVELS`, `setMapLevel`): fitBounds so each level suits any screen; moving the map by hand clears the active button. Country borders (`boundary`, admin_level 2) and country labels are style layers; place names prefer Norwegian via `coalesce`
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
- External resources: CDN links (MapLibre, font, Three.js, topojson-client, border files), Google Identity Services script, flagcdn images, and public APIs: MapTiler, Entur, Open-Meteo, OSM Nominatim, GitHub API (gist sync), Google Calendar API, World Bank (Data360, country metadata), Harvard Atlas, SSB
- No cookies, no server-side anything
