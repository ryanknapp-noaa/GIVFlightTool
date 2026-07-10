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
| **Dropsonde Spacing (70–109 NM)** | The target interval for dropsonde releases (see [Dropsonde logic](#dropsonde-release-point-logic) below). |

---

## Pattern geometry

- Waypoints are generated symmetrically around the center fix using great-circle-safe nautical-mile math (1 NM = 1 arcminute of latitude; longitude conversion uses `cos(latitude)` per row so every pass is the exact requested length at its own latitude).
- The pattern alternates: **pass → connector → pass → connector...**, starting and turning according to the selected corner.
- All legs are cardinal (due N/S/E/W) by construction when first generated, so headings and distances shown in the tables are exact, not estimates — this stays true until a waypoint is dragged (see below).
- Total waypoints = 2 × (number of passes). Total legs = (2 × passes) − 1.

## Dragging waypoints

- Every numbered waypoint marker on the map is draggable. Drag one to reshape the leg(s) on either side of it — this is the way to manually shrink or expand a specific pass or connector leg without changing the whole pattern's parameters.
- While dragging, the connected lines (and the transit line to the departure/arrival airport, if you're moving an end waypoint) follow the cursor live. Dropsonde points, tables, and exports are recalculated once you **release** the marker.
- Once you've dragged anything, an amber **"● manually edited"** badge appears above the map along with a **Reset Pattern** button, which snaps everything back to the calculated (non-dragged) positions.
- Changing any pattern parameter (center fix, pass count/length/spacing, orientation, starting corner) automatically discards manual drag edits too, since the underlying pattern itself changed.
- Dragging a waypoint off its original axis makes that leg non-cardinal (e.g. a diagonal line instead of due north/south). Distance and course calculations handle this correctly since they're computed with real great-circle math (see `legInfo()` below) — nothing in the tool assumes legs stay perfectly cardinal after a drag.

## Dropsonde release point logic

- **Every waypoint is always a drop point** — this never changes regardless of leg length or whether it's been dragged.
- **Every leg** (pass, connector, or a leg reshaped by dragging) is split into the most equal segments possible such that each segment falls between **70 and 109 NM** — e.g. a 330 NM pass becomes four equal 82.5 NM segments; an 80 NM connector stays as a single 80 NM segment (endpoints only); a leg lengthened by dragging to 150 NM becomes two 75 NM segments. This is one unified rule for the whole track, not a separate rule for passes vs. turns.
- There's a narrow, mathematically unavoidable band — roughly **109 to just under 140 NM** — where no whole number of equal segments can satisfy both the 70 NM minimum and the 109 NM maximum at once (any 2-segment split would put each half under 70 NM). In that band the tool keeps a single longer segment (over 109 NM) rather than ever letting a segment fall under 70 NM, since a too-short interval was judged the worse outcome.
- The only legs that can legitimately show a segment under 70 NM are ones shorter than 70 NM to begin with (e.g. an aggressively shortened connector, or a very tight drag) — there's no way to enforce a 70 NM minimum spacing on a leg that's physically shorter than that.
- Transit legs to/from the departure and arrival airports are **excluded** from both waypoint and drop point calculations — sondes are only released within the survey pattern itself.

## Mission distance limit

- A hard **3,200 NM** total-mission-distance limit (transit out + pattern + transit back) is checked live.
- If exceeded, a red alert banner appears with concrete options: dropping to 3 passes, shortening each pass to a computed length that would fit, or switching to one of the nearest airports in the database (each shown with its own resulting round-trip total).

---

## Map

Built on **Leaflet** with a dark CARTO basemap (loaded from a public CDN — see the note at the top of this file).

- Cyan solid lines = long survey passes; amber dashed lines = turn/connector legs and transit legs.
- Numbered circle markers = pattern waypoints — **draggable**, see [Dragging waypoints](#dragging-waypoints) above (hover for coordinates and a reminder that they're draggable).
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
| **Dropsonde List** | Fixed-width degree/minute columns (`# / lat d m / lon d m`) | Same column layout as the mission plan's DROP LOCATIONS table, just without the time column — for a dropsonde log or briefing sheet. Handles both 2- and 3-digit longitude magnitudes (e.g. Atlantic vs. Eastern Pacific basins) without breaking alignment. |
| **NHC-Style Mission Plan** | Formatted `.txt` document | Byte-for-byte layout match to the standard NOAA/NHC "Hurricane Synoptic Surveillance Mission Plan" format — fixed-width columns, degree/minute coordinates, and elapsed time-from-takeoff for each drop. Click **Download .txt** to save it. |

All coordinate math (SkyVector/ForeFlight token formats) was verified against the current public documentation for each platform at the time this tool was built; formats can change, so double-check if something doesn't paste in cleanly.

---

## Known limitations

- **Airport database** covers ~290 US large/medium airports (real coordinates, pulled from public aviation data) — enough for any realistic G-IV divert or launch base, but not literally every FAA-registered landing facility (grass strips, heliports, etc. aren't included).
- **The 70–140 NM gap**: as noted above, there's a narrow leg-length range where no drop spacing can satisfy both the 70 NM minimum and 109 NM maximum — the tool always preserves the 70 NM minimum in that case, at the cost of one longer-than-109 segment.
- **Dragging is unconstrained** — a waypoint can be dragged in any direction, not just along its original leg axis. This is intentional (real-world reshaping isn't always axis-aligned), but it means a careless drag can noticeably distort the pattern; use **Reset Pattern** to recover.
- The map's internal SVG renderer must not be overridden by any global CSS `svg{...}` rule, or polylines/drop markers will silently fail to render (this has bitten us once already — see the code comments if you're editing the CSS).
- This is a **planning aid only**. Always verify coordinates, headings, distances, and the mission plan output against current NOAA/AOC tasking, NOTAMs, and weather guidance before flight.

---

## Editing this tool

Everything — HTML, CSS, and JavaScript — lives in the single `.html` file. Key functions if you need to modify the logic:

- `generatePattern()` — builds the calculated waypoint list from the mission parameters.
- `manualWaypoints` / `onWaypointDragEnd()` / `resetManualWaypoints()` / `patternSignature()` — the drag-override state: when set, `render()` uses `manualWaypoints` instead of calling `generatePattern()`; changing a pattern parameter resets it back to `null`.
- `computeDropPoints()` / `computeSegmentCount()` — the unified 70–109 NM dropsonde placement logic described above, applied identically to every leg.
- `legInfo()` — distance/course between any two waypoints, using real great-circle math (`haversineNM()` / `bearingDeg()`) so it stays correct even after a leg is dragged off-axis.
- `renderMap()` / `initLeafletMap()` — Leaflet map drawing, including the drag/dragend handlers on each waypoint marker.
- `buildMissionPlanText()` — the NHC-style `.txt` export template.
- `decToDM()` — decimal degrees → degree/minute conversion, shared by the mission plan and Dropsonde List exports.
- `US_AIRPORTS` — the hardcoded airport array (`[ICAO, name, city, state, lat, lon]`).
