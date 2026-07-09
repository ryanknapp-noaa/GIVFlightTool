# G-IV Lawnmower Pattern Planner

A self-contained HTML tool for building NOAA G-IV synoptic surveillance ("lawnmower") flight patterns around a storm fix — computes waypoints and dropsonde release points, plots them on an interactive map, and exports everything to SkyVector, ForeFlight, AWC, and a printable NHC-style mission plan.

**File:** `g4_lawnmower_planner.html` — open it directly in any modern browser (Chrome, Edge, Firefox, Safari). No install, no server.

> Requires an internet connection. Map tiles (Leaflet/CARTO) and fonts (Google Fonts) load from public CDNs at runtime — the tool will still calculate everything offline, but the map and fonts won't render without a connection.

---

## What it does

1. You give it a storm center fix, a departure/arrival airport, and a few pattern parameters.
2. It builds a lawnmower survey pattern centered on that fix — alternating long survey passes with short turn legs.
3. It calculates dropsonde release points along that pattern.
4. It shows the whole thing on an interactive map, in data tables, and lets you export it in several formats.

Everything updates live as you change any input — there's no "generate" button.

---

## Mission Setup panel (left side)

| Field | What it does |
|---|---|
| **Mission / Storm ID** | Free text label (e.g. `INVEST 94L (AL942025)`). Shown on the map and used as the storm name in the mission plan export. |
| **Aircraft Tail #** | Tail number for the mission plan document (defaults to `N49RF`, NOAA's G-IV). |
| **Proposed Takeoff** | Free-text `DD/HHMMZ` field for the mission plan document — type it exactly as you want it to appear. |
| **Departure / Arrival ICAO** | Type an ICAO code or airport name; autocompletes against a built-in list of ~290 US airports (every large- and medium-class field — i.e. everything with a runway long enough for G-IV operations). Shown under each field once matched. |
| **Center Fix Lat / Lon** | The storm center. Enter a positive value and pick N/S or E/W from the dropdown. |
| **Number of Passes** | 3 or 4 long survey legs. |
| **Pass Length (NM)** | Length of each long survey leg (default 330 NM). |
| **Pass Spacing (NM)** | Length of the short turn/connector legs between passes (default 80 NM). |
| **Orientation** | E–W passes with N–S spacing (default), or rotate 90° for N–S passes with E–W spacing. |
| **Starting Corner** | Which corner of the pattern the aircraft enters first (NW/NE/SW/SE) — controls both the row order and the direction of the first pass. |
| **Groundspeed (kt)** | Used only for time estimates (pattern duration, mission duration, dropsonde elapsed-time column). |
| **Dropsonde Spacing (NM, min 70)** | The target interval for dropsonde releases (see [Dropsonde logic](#dropsonde-release-point-logic) below). |

---

## Pattern geometry

- Waypoints are generated symmetrically around the center fix using great-circle-safe nautical-mile math (1 NM = 1 arcminute of latitude; longitude conversion uses `cos(latitude)` per row so every pass is the exact requested length at its own latitude).
- The pattern alternates: **pass → connector → pass → connector...**, starting and turning according to the selected corner.
- All legs are cardinal (due N/S/E/W) by construction, so headings and distances shown in the tables are exact, not estimates.
- Total waypoints = 2 × (number of passes). Total legs = (2 × passes) − 1.

## Dropsonde release point logic

- **Every waypoint is always a drop point** — this never changes regardless of leg length.
- **Long pass legs** are divided evenly into the most segments possible while keeping every segment **≥ the dropsonde spacing** (default 70 NM) — e.g. a 330 NM pass at 70 NM spacing becomes four equal 82.5 NM segments, not a fixed 70-70-70-70-50 split. This guarantees no segment is ever shorter than the minimum spacing.
- **Short connector/turn legs** are treated differently:
  - Under 110 NM: **endpoints only**, no interior drop (the two waypoints are already far enough apart).
  - 110 NM or longer: **one interior drop at the exact midpoint** of the leg.
- Transit legs to/from the departure and arrival airports are **excluded** from both waypoint and drop point calculations — sondes are only released within the survey pattern itself.

## Mission distance limit

- A hard **3,200 NM** total-mission-distance limit (transit out + pattern + transit back) is checked live.
- If exceeded, a red alert banner appears with concrete options: dropping to 3 passes, shortening each pass to a computed length that would fit, or switching to one of the nearest airports in the database (each shown with its own resulting round-trip total).

---

## Map

Built on **Leaflet** with a dark CARTO basemap (loaded from a public CDN — see the note at the top of this file).

- Cyan solid lines = long survey passes; amber dashed lines = turn/connector legs and transit legs.
- Numbered circle markers = pattern waypoints (hover for coordinates).
- Pink dots = dropsonde release points (hover for sequence number and distance along track).
- Amber "D"/"A" markers = departure/arrival airports.
- Red marker = storm center fix.
- The view auto-fits to whatever's currently plotted.

## Tables

- **Waypoint Sequence** — every waypoint plus departure/arrival, with course and distance to the next point.
- **Dropsonde Release Points** — every drop point, its position, cumulative distance along the track, which leg it's on, and whether it coincides with a waypoint.

## Exports

| Card | Format | Notes |
|---|---|---|
| **SkyVector** | DMS route string (`DDMMSSN DDDMMSSW ...`) | "Open ↗" launches SkyVector centered on the fix with the route pre-filled via URL. |
| **ForeFlight** | DMS route string (`NDDMMSS/WDDDMMSS ...`) | Paste into the FPL/route editor. |
| **AWC Weather** | Decimal-degree list | "Open ↗" centers the GFA/Flight Path Tool on the fix; paste the list into its Route tool manually. |
| **Dropsonde List** | Decimal-degree list, one sonde per line | For loading into a dropsonde log or briefing sheet. |
| **NHC-Style Mission Plan** | Formatted `.txt` document | Byte-for-byte layout match to the standard NOAA/NHC "Hurricane Synoptic Surveillance Mission Plan" format — fixed-width columns, degree/minute coordinates, and elapsed time-from-takeoff for each drop. Click **Download .txt** to save it. |

All coordinate math (SkyVector/ForeFlight token formats) was verified against the current public documentation for each platform at the time this tool was built; formats can change, so double-check if something doesn't paste in cleanly.

---

## Known limitations

- **Airport database** covers ~290 US large/medium airports (real coordinates, pulled from public aviation data) — enough for any realistic G-IV divert or launch base, but not literally every FAA-registered landing facility (grass strips, heliports, etc. aren't included).
- **Course/distance display** assumes cardinal legs, which is true by construction for the pattern itself; the map's internal SVG renderer must not be overridden by any global CSS `svg{...}` rule, or polylines/drop markers will silently fail to render (this has bitten us once already — see the code comments if you're editing the CSS).
- This is a **planning aid only**. Always verify coordinates, headings, distances, and the mission plan output against current NOAA/AOC tasking, NOTAMs, and weather guidance before flight.

---

## Editing this tool

Everything — HTML, CSS, and JavaScript — lives in the single `.html` file. Key functions if you need to modify the logic:

- `generatePattern()` — builds the waypoint list from the mission parameters.
- `computeDropPoints()` — dropsonde placement logic described above.
- `legInfo()` — distance/course between two cardinal waypoints.
- `haversineNM()` / `bearingDeg()` — great-circle math used for the non-cardinal transit legs.
- `renderMap()` / `initLeafletMap()` — Leaflet map drawing.
- `buildMissionPlanText()` — the NHC-style `.txt` export template.
- `US_AIRPORTS` — the hardcoded airport array (`[ICAO, name, city, state, lat, lon]`).
