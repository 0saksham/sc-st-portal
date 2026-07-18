# Build Prompt — SC/ST Population Mapping Feature (Uttarakhand SC/ST Cell Portal)

Paste everything below to your coding agent as-is. It is written so you do not
need to make any design decisions yourself — every visual, interaction, and
data-mapping choice is specified. Two features are intentionally left to your
judgment (marked clearly at the end) — everything else is not up for
interpretation.

---

## 0. Project summary

Build a standalone, animated, 3-level drill-down data visualization feature
for the Uttarakhand SC/ST Cell government portal. It shows Scheduled
Caste (SC) and Scheduled Tribe (ST) population distribution at:

1. **State level** — all of Uttarakhand, 13 districts
2. **District level** — one district, its sub-districts (tehsils)
3. **Sub-district level** — one tehsil, its villages, searchable, individually inspectable

This is a **separate standalone page/app** for now — it is NOT linked into the
main portal's navigation yet. It must still visually look like it belongs to
the same portal.

---

## 1. Tech stack — use exactly this, no substitutions

- **Plain HTML, CSS, vanilla JavaScript.** No frameworks (no React/Vue/etc.),
  matching the parent portal (`uttarakhand-dashboard`), which is also plain
  HTML/CSS/JS using PapaParse and Chart.js.
- **Mapping:** Leaflet.js + GeoJSON layers (state/district/subdistrict
  boundaries and village points are all supplied as GeoJSON — see §2).
- **Charts:** **Apache ECharts** (not Chart.js) for every chart in this
  feature — ECharts was specifically chosen over Chart.js for its built-in,
  smoother animation system (see §6 for exactly which chart types to use
  where and what animation config to apply).
- **Routing:** hash-based client-side routing, no server/build step needed.
  Routes:
  - `#/state` — Page 1
  - `#/district/<district_code>` — Page 2
  - `#/subdistrict/<district_code>/<subdistrict_code>` — Page 3
  - Default route (`#/` or empty) redirects to `#/state`

---

## 2. Data files — exact inventory and location

All files are at: `C:\Users\saksh\OneDrive\Desktop\sc-st dash\files`

| File | Type | Purpose |
|---|---|---|
| `uttarakhand_state.geojson` | GeoJSON, 1 polygon feature | State outline + total population/SC/ST/literacy stats |
| `uttarakhand_districts.geojson` | GeoJSON, 13 polygon features | District boundaries, each with population/SC/ST/literacy stats |
| `uttarakhand_subdistricts.geojson` | GeoJSON, 111 polygon features | Sub-district (tehsil) boundaries, each with stats where available |
| `villages.geojson` | GeoJSON, 2,969 point features | Verified villages with location + full census stats (see §2.4 for why this number is smaller than the raw source files) |
| `uttarakhand_district_hq.geojson` | GeoJSON, 13 point features | District headquarters — name + lat/long, for optional map labels |
| `uttarakhand_subdistrict_hq.geojson` | GeoJSON, 111 point features | Sub-district headquarters — name + lat/long, for optional map labels |
| `uttarakhand_major_towns.geojson` | GeoJSON, 168 point features | Major towns, optional map context/labels |
| `README_geojson_data.md` | Markdown | Full data-build documentation — **read this file first**, it explains every caveat below in more depth |

All geometry is in **WGS84 (EPSG:4326 — standard lat/long)**. Do not attempt
to reproject anything; it has already been done.

### 2.1 `uttarakhand_state.geojson` — properties

```
name, households, total_population, male_population, female_population,
sc_population, st_population, literate_population,
sc_percent, st_percent, literacy_percent
```

### 2.2 `uttarakhand_districts.geojson` — properties (per feature)

```
district_code, district_name, households, total_population,
male_population, female_population, sc_population, st_population,
literate_population, sc_percent, st_percent, literacy_percent
```

### 2.3 `uttarakhand_subdistricts.geojson` — properties (per feature)

```
district_code, subdistrict_code, subdistrict_local_code, subdistrict_name,
households, total_population, male_population, female_population,
sc_population, st_population, literate_population,
sc_percent, st_percent, literacy_percent, census_2011_data_available
```

- `subdistrict_code` = the 2011 census code — **use this to filter/join villages**
- `subdistrict_local_code` = the current official tehsil code (unique per
  polygon; there are 111 polygons but only 78 distinct `subdistrict_code`
  values, because ~10 tehsils were split after the 2011 census — see §2.4)
- `census_2011_data_available` = `false` for sub-districts that didn't exist
  as separate census units in 2011. For these, still render the boundary and
  its villages normally, but if `census_2011_data_available` is `false`,
  **do not show a population share panel for that sub-district** — show a
  small inline note instead: *"2011 census data not available for this
  sub-district (created after the 2011 census)."* Village-level data (§2.4)
  is unaffected by this — villages still show correctly.

### 2.4 `villages.geojson` — properties (per feature) and important caveat

```
district_code, subdistrict_code, village_code, census_village_name,
households, total_population, male_population, female_population,
sc_population, st_population, literate_population,
sc_percent, st_percent, literacy_percent, total_workers,
matched_settlement_name
```

**Read this carefully — it affects how you present village coverage:**
Uttarakhand has ~16,793 villages in the 2011 census. This file only contains
**2,969** of them. This is intentional, not a bug: every village in this file
has been verified to have (a) authoritative census population/SC/ST data, and
(b) a real-world coordinate that has been checked to fall inside its own
correct sub-district boundary. Villages that had no reliable coordinate, or a
coordinate that landed in the wrong place, were excluded rather than shown
with wrong data. Practically:
- Some sub-districts will have many villages in the list, others very few or
  none. **This is expected — do not treat a sub-district with 0 villages in
  the list as an error.** If a sub-district's village list is empty, show:
  *"No verified village-level data available for this sub-district yet."*
- Use `census_village_name` as the display name (not `matched_settlement_name`,
  which is a secondary/matching reference only).

### 2.5 District/sub-district code reference

- `district_code` is a 3-digit zero-padded string (e.g. `"056"` = Uttarkashi).
- `subdistrict_code` is a 5-digit zero-padded string (e.g. `"00278"`).
- Always compare/filter as strings, not numbers (leading zeros matter).

---

## 3. Visual theme — must match the parent portal exactly

The base theme file `maintheme.html` (already provided to you separately) is
the single source of truth for all colors, fonts, spacing, and component
styles. Do the following:

- **Reuse its CSS custom properties directly** (`--ucost-blue`,
  `--ucost-blue-mid`, `--ucost-blue-light`, `--saffron-accent`,
  `--green-accent`, `--bg-base`, `--bg-card`, `--text-primary`,
  `--text-secondary`, `--radius`, `--shadow-sm/md/lg`, `--transition`, etc.)
  — copy the full `:root` block from `maintheme.html` verbatim.
- **Fonts:** Inter for UI text, Noto Sans / Noto Sans Devanagari as fallback
  (already linked via Google Fonts in `maintheme.html` — reuse the same
  `<link>` tags).
- **Reuse the header** (utility bar + site header + sticky nav) from
  `maintheme.html`, but **remove the main nav links** (Home / About / Contact
  / Notes / FAQ) since this page is not yet wired into the main site — replace
  the nav row with just a simple back-link/breadcrumb area (see §5).
- **Cards:** use the existing `.card` class styling (white background,
  `var(--radius)`, `var(--shadow-sm)`, hover lift with `translateY(-4px)` and
  deeper shadow) for every panel (population stats, village detail, etc.) —
  do not invent a new card style.
- **Section headers:** use the existing `.section`, `.section-header`,
  `.section-title` pattern (icon badge + title) for every major block on
  every page.
- Do **not** introduce new colors outside the existing palette. Any new UI
  element (glow effects, choropleth legend, etc.) must be built using shades/
  opacities of `--ucost-blue`, `--saffron-accent`, and `--green-accent` only.

---

## 4. Global animation & interaction language

These rules apply everywhere in this feature, not just one page. Be exact —
these values were chosen deliberately, do not approximate them.

### 4.1 Map polygon "glow" effect (used on all 3 pages differently — see below)

- Implement glow as an SVG/CSS `filter: drop-shadow(0 0 8px <color>)` layered
  with a `box-shadow`-style outer glow on the polygon's stroke, **not** just a
  fill-color change. A glow must look like light emanating from the shape's
  edge.
- Idle/default state (Page 1 only, see §5.1): a slow, subtle pulse —
  `opacity`/glow-intensity oscillating between 0.15 and 0.35 alpha, 3.2s
  duration, `ease-in-out`, infinite loop, **staggered per district** (each
  district's pulse starts 80ms after the previous one, cycling continuously,
  so the whole map has a gentle "breathing" quality rather than pulsing in
  unison).
- Hover state: glow color = `var(--saffron-accent)`, intensity jumps to full
  (alpha 0.8), transition duration 180ms `ease-out`. Simultaneously,
  stroke-width increases from 1.5px to 2.5px. A tooltip (see §4.2) appears.
- Click state: on click, before navigating, play a 220ms "confirm" pulse —
  glow flashes to alpha 1.0 then the page transitions (see §4.3) — this gives
  the user visible feedback that their click registered before the route
  changes.

### 4.2 Hover tooltip (state and district page polygons)

- Appears 120ms after hover starts (small delay to avoid flicker on fast
  mouse movement), fades in over 150ms, positioned above the cursor,
  following it.
- Content: bold name, then 2–3 stat lines (Total population, SC %, ST %),
  styled using the `.card` background/shadow/radius so it looks like a
  floating mini-card, not a native browser tooltip.
- Disappears with a 100ms fade-out on mouseleave.

### 4.3 Page transition (state → district → sub-district and back)

- On navigation, the outgoing page's map/content fades out and scales down
  slightly (0.97 scale) over 200ms `ease-in`, then the incoming page's
  content fades in and scales up from 0.97 → 1.0 over 250ms `ease-out`,
  starting immediately after the outgoing transition completes (total ~450ms
  perceived transition). Do not use a hard cut or a full-page reload — this is
  a single-page app, manage this entirely in JS/CSS.

### 4.4 Entrance animation (every time a map page first loads)

- The relevant boundary polygon(s) (state outline on Page 1, sub-district
  boundaries on Page 2, the single sub-district boundary on Page 3) draw in
  using a stroke-dasharray "draw" animation (like a pen tracing the outline),
  600ms, `ease-in-out`.
- Immediately after the outline finishes drawing, the fill fades in (opacity
  0 → target) over 300ms.
- If there are multiple child polygons inside (districts on Page 1,
  sub-districts on Page 2), each one fades/scales in (opacity 0→1, scale
  0.9→1) with an 80ms stagger between each, starting right after the parent
  outline's fill-fade completes.
- Numeric stats (population counters — see §4.5) begin counting up only after
  all polygon entrance animation is complete, not simultaneously.

### 4.5 Animated number counters

- Any raw number shown in a stat panel (total population, SC population,
  etc.) must count up from 0 to its final value over 900ms using an
  `ease-out` easing curve (fast start, slow finish), not a linear count. Use
  `toLocaleString()` formatting (comma thousand-separators) at every
  intermediate frame, not just the final value.

---

## 5. Page 1 — State Level (`#/state`)

### 5.1 Header/layout

- Reused portal header (see §3), title area: "SC/ST Population Distribution —
  Uttarakhand" as an `<h1>`.
- Below the header: a two-column layout on desktop (map ~65% width, stats
  sidebar ~35% width, using the existing `.page-layout` grid pattern from
  `maintheme.html`), stacking to single-column below 1024px (reuse the
  existing responsive breakpoint behavior).

### 5.2 Map (left/main column)

- Render `uttarakhand_state.geojson` as the outer outline (static, no
  interaction).
- Render `uttarakhand_districts.geojson` on top — 13 clickable/hoverable
  polygons.
- Apply the entrance animation from §4.4, then the idle pulse-glow from §4.1
  once entrance completes.
- On hover: glow + tooltip per §4.1/§4.2, tooltip shows district name, total
  population, SC %, ST %.
- On click: confirm-pulse (§4.1) then navigate to `#/district/<district_code>`
  using the transition in §4.3.
- **Choropleth toggle**: place a small segmented control above the map with
  three options: **"Flat" / "SC intensity" / "ST intensity"**, default =
  "Flat" (uses the theme's normal district fill colors). When "SC intensity"
  or "ST intensity" is selected, recolor every district polygon's fill using
  a single-hue intensity scale (light → dark) based on `sc_percent` or
  `st_percent` respectively — use `--ucost-blue-light` → `--ucost-blue-dark`
  as the low→high scale for SC intensity, and `--green-light` → `--green-accent`
  (darkened via CSS `filter: brightness()`) for ST intensity. Include a small
  gradient legend (min/max values labeled) below the toggle when either
  intensity mode is active. Switching modes animates the fill-color
  transition over 400ms `ease-in-out` (do not hard-swap colors).

### 5.3 Population share panel (right/sidebar column)

- A `.card` titled "Uttarakhand — Overall SC/ST Population" containing:
  - 3 animated counters (§4.5) side by side: Total Population, SC Population
    (+ `sc_percent`% shown next to it), ST Population (+ `st_percent`%).
  - An ECharts **donut chart** (see §6.1) showing SC / ST / Other population
    share.
- Below that, a second `.card` titled "Districts Ranked by SC/ST Concentration"
  containing an ECharts **horizontal bar chart** (see §6.2) ranking all 13
  districts by combined `sc_percent + st_percent`, sorted descending. Hovering
  a bar highlights (glows) the corresponding district polygon on the map, and
  vice versa (hovering a district polygon highlights its bar) — this
  cross-highlight interaction is required, not optional.

### 5.4 Tribal communities section — SKIP for this build

Do not build the tribal communities section in this pass. The user will
provide tribal community content after the main project is built. Leave a
commented-out placeholder in the code (`<!-- TODO: tribal communities section,
content to follow -->`) at the location on the page where it will eventually
go (directly below the population share panel, full width), so it's easy to
slot in later. Do not build any UI for it now.

### 5.5 Schemes section — DO NOT BUILD

This was in earlier planning but has been dropped from scope entirely. Do not
build any schemes-related UI, placeholder, or data structure.

### 5.6 Events section — DO NOT BUILD (future update)

Events are deferred to a future update. Do not build any events UI or
placeholder for this pass.

---

## 6. Chart specifications (ECharts) — exact chart type per data point

Do not substitute chart types. Each was chosen for what it needs to
communicate, per the project's own postmortem on why the last attempt's
charts "looked random."

### 6.1 Donut chart (SC / ST / Other population share)

- Used on: Page 1 sidebar, Page 2 district panel, Page 3 sub-district panel.
- ECharts `pie` series with `radius: ['55%', '75%']` (donut, not full pie).
- Center label: total population number (animated counter, §4.5) with "Total
  Population" sublabel, placed in the donut's hollow center using ECharts
  `graphic` or a centered absolute-positioned label.
- Segments: SC (`--ucost-blue`), ST (`--green-accent`), Other/remaining
  (`--border-strong` light gray).
- Animation: `animationType: 'scale'`, `animationEasing: 'elasticOut'`,
  `animationDuration: 900`, segments should appear to "grow out" from the
  center, not just fade in.
- Legend below the chart, using theme colors/fonts.

### 6.2 Horizontal ranking bar chart (districts ranked by SC/ST concentration)

- Used on: Page 1 sidebar only.
- ECharts horizontal `bar` series, sorted descending by combined SC+ST %.
- Bars colored with a gradient from `--ucost-blue-light` to `--ucost-blue`,
  using ECharts' `linearGradient` fill.
- `animationEasing: 'cubicOut'`, `animationDelay` staggered per bar (index *
  60ms) so bars grow left-to-right in sequence, not all at once.
- Clicking or hovering a bar cross-highlights the map polygon (§5.3).

### 6.3 Scatter chart (literacy vs. SC/ST population) — sub-district and village detail levels only

- Used on: Page 3 sub-district panel (all villages in that sub-district
  plotted at once) and again, highlighted, in the village detail panel when
  one village is selected.
- X-axis: `literacy_percent`, Y-axis: `sc_percent + st_percent` combined,
  point size scaled by `total_population`.
- Points colored `--ucost-blue-light` at 60% opacity by default; the
  currently-selected village (if any) renders as a larger point in solid
  `--saffron-accent` with a glow (reuse §4.1's glow technique), on top of the
  others.
- This chart directly supports the "click a village updates data in place"
  requirement (§8) — when a village is selected from the search/list, this
  chart re-renders to highlight that point without a full chart teardown/
  rebuild (use ECharts' `setOption` with `notMerge: false` for a smooth
  transition, not a full chart destroy+recreate).

### 6.4 Village detail micro-charts (shown only once a village is selected, Page 3)

For the selected village, show 3 small charts side-by-side in the village
detail card:
- A small donut (same style as §6.1, scaled down) for that village's SC/ST/
  Other split.
- A small horizontal bar comparing that village's `sc_percent`/`st_percent`
  against the parent sub-district's average (2 bars per category — village
  vs. sub-district average) — this gives the user immediate context for
  whether this village is above/below the local average.
- A simple animated horizontal "gauge"-style bar for `literacy_percent`
  (0–100 scale, single bar, filled with a gradient, animated width 0→value
  over 700ms `ease-out`).

All village-detail charts animate in with a 400ms fade+scale-up (0.95→1)
whenever a new village is selected, replacing the previous village's charts —
this should feel like the panel is "refreshing," not blinking/flashing.

---

## 7. Page 2 — District Level (`#/district/<district_code>`)

- Reused header, same two-column layout as Page 1.
- Breadcrumb below the header: "Uttarakhand › [District Name]" — "Uttarakhand"
  is a clickable link back to `#/state`.
- **Map:** render the state outline dimmed/faded in the background for
  context (10–15% opacity, non-interactive), with the current district's
  boundary emphasized (from `uttarakhand_districts.geojson`, filtered to this
  `district_code`), and all its sub-district polygons on top (from
  `uttarakhand_subdistricts.geojson`, filtered to this `district_code`),
  fully interactive.
- Sub-district polygons get the **exact same entrance animation, idle glow,
  hover glow, tooltip, and click-to-navigate behavior as districts did on
  Page 1** (§4.1–§4.3) — this is the fix for the previous version's failure,
  where sub-districts were "just dots." They must be real boundary shapes
  with the same animation quality as the district level, not a downgrade.
- Clicking a sub-district navigates to
  `#/subdistrict/<district_code>/<subdistrict_code>` (use the
  `subdistrict_code`, i.e. the 2011 census code, not `subdistrict_local_code`,
  in the URL — this is what §2.3 and villages.geojson join on).
- **Sidebar:** same panel pattern as §5.3 but scoped to this district: donut
  chart (§6.1) for this district's SC/ST split, animated counters, and a
  smaller version of the ranking bar chart (§6.2) showing where this district
  ranks among all 13 (highlight this district's own bar distinctly).
- If a sub-district polygon has `census_2011_data_available: false`, style it
  with a subtle diagonal-hatch pattern fill (in addition to normal coloring)
  so it's visually distinguishable, and its tooltip should say "2011 census
  data not available" instead of showing stats.

---

## 8. Page 3 — Sub-district Level (`#/subdistrict/<district_code>/<subdistrict_code>`)

- Reused header, breadcrumb: "Uttarakhand › [District Name] › [Sub-district
  Name]", both prior levels clickable.
- **Map:** render only this one sub-district's boundary (from
  `uttarakhand_subdistricts.geojson`, filtered to this
  `district_code`+`subdistrict_code`), zoomed/fit to it.
  - **No hover-glow, no click interaction on the boundary shape itself** —
    it is static once its entrance animation (§4.4) completes. This is
    intentional, confirmed by the project owner — do not add hover behavior
    here even though earlier levels have it.
  - **Do not plot any village points by default.** The map starts empty of
    village markers.
- **Village search + list** (place this in the sidebar, above or below the
  stats panel — your call on order, but it must be visible without scrolling
  on a standard desktop viewport):
  - A search input (reuse `.header-search` input styling from the theme) that
    filters the village list live as the user types, matching against
    `census_village_name`.
  - Below it, a scrollable list of all villages in this sub-district (from
    `villages.geojson`, filtered by `district_code` + `subdistrict_code`).
    Each list item shows village name + total population, styled as a
    compact row (reuse `.dropdown-link`-style hover treatment from the
    theme's nav dropdown for consistency).
  - If the filtered/full list is empty, show: *"No verified village-level
    data available for this sub-district yet."* (per §2.4).
- **Selecting a village** (via search-then-click, or directly clicking a list
  item):
  - **Do not navigate to a new page or change the URL route.** This happens
    in place.
  - The map places a single marker at that village's coordinates, styled with
    a glow (§4.1 technique, `--saffron-accent`) and a small pulsing "ping"
    ring animation (expanding, fading circle, 1.4s loop) so it's immediately
    findable on the map. If a different village was previously selected, its
    marker is removed (with a 200ms fade-out) as the new one appears.
  - The map auto-pans/zooms smoothly (Leaflet's `flyTo`, ~600ms) to center on
    the newly-selected village, while staying within the sub-district
    boundary's extent (don't zoom in so far that the boundary context is
    lost).
  - A **village detail card** appears/updates below or beside the map (per
    §4.4/§6.4 animation rules — fades and scales in, doesn't hard-replace):
    village name, households, total population, male/female split,
    literate population, total workers, and the three micro-charts from
    §6.4, plus the scatter chart from §6.3 re-rendered with this village
    highlighted.
  - A **"Download village data"** button on this card (see §9 — you have
    design freedom on this one).
- **Sub-district population panel:** same donut+counters pattern as §5.3/§7,
  scoped to this sub-district. If `census_2011_data_available` is `false` for
  this sub-district, replace this whole panel with the inline note specified
  in §2.3.

---

## 9. Two features left to your design judgment

Everything above is fully specified. These two are intentionally not — design
and implement them using your own best judgment for UX and animation quality,
consistent with everything else in this spec (same theme, same animation
language from §4):

1. **"Compare districts"** — some way for a user on Page 1 to compare two or
   more districts against each other (data table, side-by-side charts,
   overlay on the ranking bar chart — your call).
2. **"Download village data"** button (Page 3, §8) — decide the export format
   (CSV/JSON/print-friendly view) and exact placement/styling.

---

## 10. Explicit non-goals for this build (do not implement)

- Schemes section (dropped from scope — §5.5)
- Events section (deferred to a future update — §5.6)
- Tribal communities section (deferred until after main build — §5.4)
- Hindi translation of village/sub-district/district names from the data
  files (per earlier project decision, only major UI chrome gets translated,
  not this feature's data-heavy content)
- Linking this page into the main portal's navigation (still standalone)
