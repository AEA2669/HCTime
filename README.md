# HCTime — Operations Command Center

A suite of real-time aviation operations dashboards built with pure HTML, CSS, and vanilla JavaScript. Designed for airport ramp/ground operations teams to monitor time-critical schedules, live weather conditions, and safety alerts on always-on displays.

---

## Project Overview

**Idea:** Replace manual weather checks and wall clocks with a single browser-based command center that auto-updates, never sleeps, and visually escalates when action is needed. Every page is a self-contained `.html` file — no build step, no framework, no server required. Open it in a browser and it runs.

**Tech Stack:**
- **HTML5 / CSS3 / Vanilla JS** — zero dependencies, zero build tools
- **CSS Custom Properties** — a shared dark-theme design system (`--bg-primary`, `--accent`, `--alert-*`, etc.)
- **Glassmorphism UI** — `backdrop-filter: blur()`, translucent cards, layered radial-gradient background orbs with keyframe drift animations
- **Google Fonts** — Inter (UI text) + JetBrains Mono (data/monospace values)
- **Aviation Weather API** — live METAR/TAF data from `aviationweather.gov/api/data/metar`
- **CORS Proxy Fallback Chain** — tries direct fetch, then cycles through `allorigins`, `corsproxy.io`, `codetabs` to survive cross-origin restrictions
- **Stability Watchdogs** — auto-reload on stale data, error accumulation, and periodic hard refresh to keep kiosk displays running 24/7

---

## Pages

### 1. `index.html` — Operations Command Center (Main Dashboard)

**Idea:** The primary hub. A two-panel layout showing time countdowns on the left and a full weather overview of 11 airports on the right.

**Layout:**
- **Left Panel (33vw):** Two stacked countdown cards — "Field Conditions" schedule at 00:00, 06:00, 12:00, 18:00.
- **Right Panel:** A hero weather card for DFW (KDFW) with full METAR detail, plus a 10-card grid showing live conditions for CLT, ORD, MIA, LAX, JFK, LHR, AKL, PVG, FCO, and DEN.

**Logic:**
- **Countdown Engine (`updateSection`):** Given a schedule array (e.g., `[[0,0],[6,0],[12,0],[18,0]]`), calculates the next upcoming event, time remaining, and an elapsed ratio. Drives an SVG ring (stroke-dashoffset animation), a progress bar, and a pip timeline where completed events turn green and the current one pulses amber.
- **Alert State:** When the countdown drops below 10 minutes, the card transitions to an amber pulse animation, ring/bar change color, and text glows — a visual "heads up."
- **Weather Fetch (`fetchWx`):** Polls `aviationweather.gov` every 5 minutes for all 11 ICAO stations in one batch call. Uses a proxy fallback chain with `AbortController` timeouts (15s per attempt). On success, builds a lookup map by ICAO and pushes values into the DOM.
- **METAR Parsing:** Extracts temp (C→F conversion), dewpoint, wind direction/speed/gusts, visibility, ceiling (scans for lowest BKN/OVC layer), altimeter (mb→inHg), humidity (Magnus formula), flight category (color-coded badges: VFR green, MVFR blue, IFR red, LIFR magenta), and a weather icon derived from the wx string.
- **Color-Coding:** Wind values turn amber at 30kt / red at 35kt. Visibility turns amber < 5SM / red < 3SM. Ceiling turns amber < 1000ft / red < 500ft.
- **Staleness Detection:** If no successful fetch for 15 min, a "WX: STALE" warning appears. After 68 minutes of no updates, auto-redirects to `Clocks.html` as a failover. Hard page refresh every 40 minutes regardless.
- **Settings Panel:** Persisted to `localStorage` — toggle scanlines, background animation, hover effects, color-coding, category accent borders, METAR display, clock format, poll interval, UI scale, and brightness.

---

### 2. `Clocks.html` — World Times & Countdowns

**Idea:** A fallback/companion display focused on time. Shows local + UTC hero clocks, two countdown timers (SIMBA and Field schedules), and a 10-city world clock grid.

**Layout:**
- **Top Row:** Two hero clock cards — Local time and UTC, each with date.
- **Bottom Left:** Two countdown sections (SIMBA schedule: 05:00, 08:00, 11:00, 14:00, 16:30, 20:00; Field schedule: 00:00, 06:00, 12:00, 18:00) with identical ring/pip/bar logic to `index.html`.
- **Bottom Right:** 10 world clock tiles (JFK, LHR, MAD, DOH, SOF, NRT, AKL, LAX, KOA, DEN) using `Intl.DateTimeFormat` with IANA timezone names.

**Logic:**
- **`tick()` runs every 1 second** — updates both hero clocks, both countdown sections, and all 10 world time tiles.
- **World Clocks:** Uses `Intl.DateTimeFormat` with `formatToParts()` to extract hours/minutes/seconds in each timezone, plus `timeZoneName: 'shortOffset'` for the UTC offset label.
- **No weather fetch** — this is a pure time display, making it the ideal failover page.
- **Stability:** Auto-reloads after 12 hours. Reloads after 3 uncaught errors. Re-ticks immediately on `visibilitychange`.

---

### 3. `dfw.html` — DFW Weather Intelligence

**Idea:** A deep-dive weather page for Dallas/Fort Worth International (KDFW, field elevation 607 ft). Full METAR decoding plus TAF forecast.

**Layout:**
- **Hero Card:** Large weather icon, flight category badge, temperature (°C), dewpoint, humidity bar, and weather description string.
- **Wind Card:** Compass rose with animated needle, wind direction/speed/gusts in knots and MPH, variable wind indicator.
- **Visibility & Ceiling Cards:** Big number readouts with SM/km conversion and AGL ceiling height.
- **Cloud Layers Table:** Dynamically built rows — cover type (CLR/FEW/SCT/BKN/OVC with color badges), base altitude, and type.
- **Pressure Card:** Altimeter in inHg, hPa, mmHg, and sea-level pressure.
- **Performance Card:** Density altitude and pressure altitude calculated from field elevation + altimeter setting + temperature. Temp/dewpoint spread.
- **Raw Data Cards:** Full raw METAR and TAF strings with issuance timestamps.

**Logic:**
- **`fetchMetar()`** — Polls KDFW METAR every 5 min via proxy fallback. Parses the JSON response and calls `applyMetar()` to populate ~30 DOM elements.
- **`fetchTaf()`** — Polls KDFW TAF on the same interval. Displays raw TAF text.
- **`calcDensityAlt(tempC, altimInHg)`** — Computes pressure altitude (`fieldElev + (29.92 - altim) * 1000`) and density altitude using ISA deviation (`pressAlt + 120 * (temp - isaTemp)`).
- **`calcRH(t, td)`** — Magnus formula for relative humidity from temp and dewpoint.
- **`describeWx(wxString)`** — Maps raw METAR weather codes (e.g., `+TSRA`, `FZRA`, `BR`) to human-readable descriptions via a lookup table.
- **Compass Needle:** CSS `transform: rotate(Xdeg)` driven by `wdir` from the METAR.
- **Stability:** Same watchdog pattern — 12h auto-reload, error counter, visibility-change re-fetch.

---

### 4. `clt.html` — CLT Weather Intelligence

**Idea:** Identical deep-dive weather page for Charlotte Douglas International (KCLT, field elevation 748 ft).

**Layout & Logic:** Structurally identical to `dfw.html`. The only differences are:
- `STATION = 'KCLT'`
- `FIELD_ELEV_FT = 748`
- Page title and ICAO header say "CLT / Charlotte Douglas Intl"

All the same METAR/TAF parsing, density altitude calculations, compass rose, cloud layer table, and proxy fallback logic apply.

---

### 5. `High Winds.html` — High Winds Alert (25+ Knots)

**Idea:** A full-screen alert overlay displayed when sustained winds reach 25+ knots. Designed to be unmissable on operations screens.

**Layout:**
- **Hero Banner:** Pulsing cyan-bordered card with animated hazard stripe background, large wind icon, "HIGH WINDS" text, subtitle "Sustained winds 25+ knots", and an elapsed duration timer.
- **Below:** Same weather detail cards as `dfw.html` — conditions hero, wind compass, visibility, ceiling, clouds, raw METAR/TAF.

**Logic:**
- **Duration Timer (`tickAlertTimer`):** Counts up from the moment the page was opened, showing how long the high wind condition has been active.
- **Hazard Animation:** CSS `background: linear-gradient(135deg, ...)` with `background-size: 40px` and a `@keyframes hazardMove` that shifts the pattern continuously — creating a moving hazard-stripe effect.
- **Banner Pulse:** `@keyframes bannerPulse` oscillates the `box-shadow` glow intensity every 3 seconds.
- **Weather data** is fetched identically to `dfw.html` (KDFW, 5-min poll, proxy fallback).
- Accent color scheme uses `--accent-cyan (#22d3ee)` instead of the default blue.

---

### 6. `Ramp Closed.html` — Ramp Closed Alert

**Idea:** A full-screen critical alert displayed when the ramp is closed (lightning, severe weather, etc.). Red-themed for maximum urgency.

**Layout:**
- **Hero Banner:** Pulsing red-bordered card with animated hazard stripes, stop-hand icon, "RAMP CLOSED" text, and an elapsed closure duration timer.
- **Below:** Full weather detail cards identical to the `dfw.html` layout.

**Logic:**
- **Ramp Timer (`tickRampTimer`):** Counts up from page open, tracking total ramp closure duration.
- **Color Scheme:** Uses `--accent-red (#f87171)` throughout — background orbs, banner border, corner accents, title text all shift to red tones.
- Hazard stripe animation and banner pulse work the same as `High Winds.html`, recolored red.
- Weather fetch targets KDFW with the standard 5-min poll cycle.

---

### 7. `Yellow Alert.html` — Yellow Alert (Lightning Within 10 Miles)

**Idea:** A warning-level alert displayed when lightning is detected within 10 miles. Amber/gold-themed, less critical than Ramp Closed but still demands immediate attention.

**Layout:**
- **Hero Banner:** Pulsing amber-bordered card with amber hazard stripes, lightning bolt icon, "YELLOW ALERT" text, subtitle "Lightning within 10 miles", and an elapsed alert duration timer.
- **Below:** Full weather detail cards identical to `dfw.html`.

**Logic:**
- **Alert Timer (`tickAlertTimer`):** Counts up from page open to track how long the yellow alert has been active.
- **Color Scheme:** Uses `--accent-amber (#fbbf24)` — background orbs, banner, corner accents, title all shift to amber/gold.
- Weather fetch is identical to the other alert pages (KDFW, 5-min poll).

---

## Shared Architecture & Patterns

### Design System
All pages share a common CSS custom property palette defined in `:root`. The dark theme (`#060a14` background) with glassmorphism cards, animated background mesh/orbs, and subtle scanline overlays creates a consistent "command center" aesthetic.

### Weather Fetch Pipeline
```
aviationweather.gov API
        │
        ▼
  Try direct fetch (works on localhost)
        │ fail?
        ▼
  Try allorigins.win proxy
        │ fail?
        ▼
  Try corsproxy.io proxy
        │ fail?
        ▼
  Try codetabs proxy
        │ fail?
        ▼
  Increment error counter → reload after 10 consecutive failures
```

### Stability Features
| Feature | Value | Purpose |
|---|---|---|
| Auto-reload | 12h (alert pages) / 40min (index) | Prevent memory leaks on kiosk displays |
| Error counter | 3 uncaught errors → reload | Self-heal from runtime crashes |
| Stale detection | 15 min no data → "STALE" warning | Alert operators to feed issues |
| Failover redirect | 68 min no updates → `Clocks.html` | Graceful degradation if API is completely down |
| Visibility change | Re-fetch on tab focus | Instant update when returning from another tab |
| Fetch timeout | 15–30s per proxy attempt | Don't hang on slow/dead proxies |

### Key Calculations
- **Temperature:** Celsius to Fahrenheit (`C * 9/5 + 32`)
- **Humidity:** Magnus formula (`100 * exp((a*Td)/(b+Td)) / exp((a*T)/(b+T))` where a=17.625, b=243.04)
- **Density Altitude:** `pressAlt + 120 * (tempC - isaTemp)` where `pressAlt = fieldElev + (29.92 - altim) * 1000` and `isaTemp = 15 - pressAlt * 0.00198`
- **Ceiling:** Lowest cloud layer with BKN or OVC coverage
- **Wind conversion:** Knots to MPH (`kt * 1.151`)

---

## How to Run

1. Open any `.html` file directly in a modern browser (Chrome, Edge, Firefox).
2. No server, no install, no build step required.
3. For kiosk/always-on use: open `index.html` in full-screen (F11). The page self-maintains via auto-reload and failover logic.
4. Navigate between pages using the back links in the top bar, or open alert pages directly when conditions warrant.

## File Map

| File | Role |
|---|---|
| `index.html` | Main dashboard — countdowns + 11-airport weather grid |
| `Clocks.html` | Time-only display — local/UTC + world clocks + countdowns |
| `dfw.html` | Deep weather intelligence for DFW (KDFW) |
| `clt.html` | Deep weather intelligence for CLT (KCLT) |
| `High Winds.html` | Alert overlay — sustained winds 25+ knots |
| `Ramp Closed.html` | Critical alert overlay — ramp closure |
| `Yellow Alert.html` | Warning overlay — lightning within 10 miles |
