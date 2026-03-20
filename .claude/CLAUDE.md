# CLAUDE.md — DJI Mini 4 Pro Flight Planner

## Project Overview

Single-file browser app for planning DJI Mini 4 Pro waypoint missions. No build step, no server, no install.

- **Live URL**: https://vviberoptics.github.io/flight-planner/
- **Source**: `Flight Planner.html` — edit this file, copy to `index.html`, push
- **Size**: ~5000 lines of HTML/CSS/JS in one file
- **Deployment**: GitHub Pages serving `index.html` from `master`

## Architecture

```
Flight Planner.html
  <style>   CSS design system + responsive layout
  <body>    HTML structure (toolbar, map div, sidebar)
  <script>  JS modules as IIFEs registered on window:
    Section  0  Bus          Event bus (on/off/emit)
    Section  1  Constants    CAM specs, FT2M, limits
    Section  2  AppState     State manager + change events
    Section  3  Geo          Math: ll2m, m2ll, pip, polyArea, distance, bearing, resample
    Section  4  Planner      Plan generators: ortho, corridor, orbit, perimeter
    Section  5  Obstacles    Waypoint filtering + smoothing near obstacle zones
    Section  6  Exporter     DJI KMZ (WPML), CSV, KML, GeoJSON, download helper
    Section  7  SrtParser    DJI .SRT subtitle parser + trajectory overlay
    Section  8  AltChart     Chart.js altitude profile sparkline
    Section  9  MapCtrl      Leaflet map init, draw mode, vertex handles, rotation handle
    Section 10  UI           Tab/mode switching, results, toasts, file uploads, event wiring
    Section 11  Search       Nominatim geocoder + GPS locate
    Section 12  Init         Keyboard shortcuts + DOMContentLoaded bootstrap
    Bridge       Bus.on('plan:regen') -> Planner.generate()
```

All modules are IIFEs. Globals: `window.Bus`, `window.AppState`, `window.Geo`, `window.Planner`, `window.Obstacles`, `window.Exporter`, `window.AltChart`, `window.Search`, `window.SrtParser`, `window.MapCtrl`, `window.UI`.

## Module Reference

### Bus
Event bus. Decouples all modules.
- `Bus.on(event, fn)` — subscribe
- `Bus.off(event, fn)` — unsubscribe
- `Bus.emit(event, data)` — fire (errors caught per-listener, logged to console)

### AppState
Single source of truth. Emits `state:{key}` on every `set()`.
- `AppState.get(key)` — read value
- `AppState.set(key, value)` — write + emit `state:{key}`
- `AppState.reset()` — restore defaults + emit all keys + `state:reset`

State keys: `phase`, `mode`, `drawMode`, `siteCoords`, `orbitCenter`, `gridAngle`, `flightPlan`, `obstacles`, `obstacleMode`, `trajectoryData`, `photoData`

### Geo
All spatial math. No external dependencies.
- `ll2m(ref, p)` — project `{lat,lng}` to `{x,y}` meters from ref (equirectangular)
- `m2ll(ref, x, y)` — inverse of ll2m
- `pip(px, py, polyXY)` — point-in-polygon (ray cast, projected space)
- `polyArea(coords)` — area m² via Shoelace after projection
- `polyCenter(coords)` — centroid `{lat,lng}`
- `bearing(a, b)` — 0–360 clockwise from north
- `distance(a, b)` — Haversine metres
- `resample(pts, spacing)` — evenly spaced polyline interpolation
- `farthestPair(coords)` — indices [i, j] with max pairwise distance
- `offsetEdge(a, b, dist, side)` — perpendicular edge offset
- `rotatePoint(p, center, angle)` — 2D rotation in projected space

### Planner
All generators read DOM inputs directly. Returns plan object with `.wps` array.

**Waypoint object shape:**
```js
{ lat, lng, alt_ft, heading, gimbal, zoom, hoverTime, action, line }
```
- `action`: `'takePhoto'` | `'record'` | `'hover'`
- `heading`: degrees 0–360 (null = followWayline)
- `gimbal`: degrees, typically -90 (nadir) to 0

**Plan object shape:**
```js
{ wps, mode, dist, time, interval, area, lines, stats: { ... } }
```

- `Planner.ortho(coords, params)` — lawnmower grid; rotatable via `gridAngle`
- `Planner.corridor(coords, params)` — ortho with `gridAngle` auto-set to farthest-pair bearing
- `Planner.orbit(center, params)` — circle around point; multi-altitude passes (100%/70%/40%)
- `Planner.perimeter(coords, params)` — offset edge survey; inward or outward per `direction`
- `Planner.generate()` — reads `AppState` + DOM inputs, calls appropriate generator, sets `flightPlan`, emits `plan:generated`

**Constants:**
- `MAX_WPS = 9999` — application-level cap (returns null if exceeded)
- `DJI_SPLIT = 1999` — KMZ splits into separate `.wpml` files above this
- `BATTERY_MIN = 25` — usable minutes per charge
- `DRONE_ENUM = 68` — Mini 4 Pro value in DJI WPML schema

### Obstacles
- `Obstacles.apply(wps, obstacles)` — filter waypoints inside obstacle circles + 3-pass smoothing near boundaries
- `Obstacles.smooth(wps, windowSize)` — sliding-window altitude smoother

### Exporter
- `Exporter.djiKmz(plan, speed)` — builds DJI WPML inside a ZIP. Splits at `DJI_SPLIT`. Uses `navigator.share` on iOS Safari, anchor-download otherwise.
- `Exporter.csv(plan)` — columns: WP, Lat, Lng, Alt_ft, Alt_m, Heading, Gimbal, Line
- `Exporter.kml(plan)` — LineString KML, altitude relativeToGround
- `Exporter.geoJson(plan)` — FeatureCollection with LineString + per-waypoint Points
- `Exporter.download(content, filename, mimeType)` — public download helper

### AltChart
- `AltChart.update(plan)` — draws/redraws Chart.js sparkline in `#altitudeProfile`. Guard: `if (typeof Chart === 'undefined') return`. Destroys previous chart instance before creating new one.

### MapCtrl
Leaflet wrapper. Layers: `polyLayer` (polygon), `gridLayer` (flight path), `wpLayer` (waypoint dots), `handleLayer` (rotation + vertex).

Map tiles:
- Satellite: ArcGIS World Imagery + CartoDB light labels overlay
- Street: CartoDB dark_all

- `MapCtrl.init()` — creates map centered at `[35.1, -106.6]` (New Mexico), wires click/dblclick
- `MapCtrl.startDraw()` — toggle crosshair cursor, capture clicks
- `MapCtrl.finishDraw()` — called on dblclick or click-near-first-point (< 18px desktop / 36px touch)
- `MapCtrl.cancelDraw()` — Escape key or second click on draw button
- `MapCtrl.setPolygon(coords)` — sets siteCoords, redraws, emits `plan:regen`
- `MapCtrl.drawPlan(plan)` — renders path polyline + waypoint circles; orbit/perimeter get heading indicator lines
- `MapCtrl.toggleLayer()` — satellite ↔ street
- `MapCtrl.clearAll()` — clears all layers + `AppState.reset()`
- `MapCtrl.fitBounds()` — pads 30%
- `MapCtrl.getMap()` — returns raw Leaflet map instance

Path colors: ortho/corridor = `#ffab40` (orange), orbit/perimeter = `#e040fb` (purple). First wp = green, last = red.

### UI
- `UI.init()` — wires all DOM event listeners; call once on DOMContentLoaded
- `UI.switchTab(phase)` — `'plan'` | `'analyze'`
- `UI.setMode(mode)` — switches param panel `.visible` class, updates draw button label, emits `mode:changed`
- `UI.updateResults(plan)` — populates stat cards, warnings, calls `AltChart.update`
- `UI.updateGsdAdvisory()` — reads orthoAlt/orthoPhoto/orthoGsd inputs, updates `#gsdAdvisory` with color-coded feedback
- `UI.toast(message, type, duration)` — types: `'info'` | `'success'` | `'warning'` | `'error'`
- `UI.handleKmlUpload(file)` — supports KML and KMZ (extracts with JSZip). Multi-strategy KML parser.
- `UI.handleSrtUpload(file)` — parses .SRT, calls `SrtParser.overlay`, updates deviation stats
- `UI.handlePhotoUpload(files)` — reads EXIF GPS from JPEGs via `DataView` parser, plots coverage markers

### Search
- `Search.init(map)` — wires `#searchInput` to Nominatim API (400ms debounce, 6 results)
- `Search.locate()` — `navigator.geolocation`, temp `circleMarker` for 8s

### SrtParser
Parses DJI `.SRT` subtitle files (block format with `[latitude:]`, `[longitude:]`, `[rel_alt:]` tags).
- `SrtParser.parse(text)` — returns `[{time_s, lat, lng, alt_m, yaw, pitch}]`
- `SrtParser.overlay(map, trajectoryData, plannedWps)` — draws colored polyline (gradient by altitude), dashed planned path, red circles for deviations > 10m. Returns `{maxDev, avgDev, rmse, coveragePct, actualTime}`.
- `SrtParser.clear(map)` — removes overlay layer group

## Key Event Flows

### Draw polygon to plan
```
User clicks map (drawMode=true)
  -> MapCtrl._onMapClick() accumulates drawPts
  -> dblclick or click-near-start -> MapCtrl.finishDraw()
  -> MapCtrl.setPolygon(coords) -> AppState.set('siteCoords')
  -> Bus.emit('plan:regen')
  -> Planner.generate()
  -> AppState.set('flightPlan', result)
  -> Bus.emit('plan:generated', result)
  -> MapCtrl.drawPlan(result) + UI.updateResults(result)
```

### Parameter change to plan update
```
User changes any param input/slider
  -> UI.debounceRegen() (200ms)
  -> Bus.emit('plan:regen')
  -> [same as above from Planner.generate()]
```

### Mode switch
```
User clicks mode button
  -> UI.setMode(mode)
  -> AppState.set('mode', mode)
  -> param panel .visible toggled
  -> Bus.emit('mode:changed', mode)
  -> UI.debounceRegen() -> plan:regen
```

### Rotation handle drag
```
MapCtrl.rotHandle.on('drag')
  -> AppState.set('gridAngle', snapped)
  -> MapCtrl._updateRotationHandle()
  -> Bus.emit('plan:regen')
```

### KMZ export chain
```
User clicks #exportKmz
  -> UI._wireExports handler
  -> reads speed from current mode's speed input
  -> Exporter.djiKmz(plan, speed)
  -> _buildWpmlDoc() wrapping _buildWaylineFolder()
  -> JSZip.generateAsync() -> Exporter.download()
  -> iOS: navigator.share | else: anchor click
```

## DOM Element ID Conventions

**Toolbar buttons** (`btnXxx`):
`btnDrawArea`, `btnUploadKml`, `btnObstacles`, `btnLayers`, `btnClear`, `btnLocate`, `btnToggleSidebar`

**Tab controls**: `tabPlan`, `tabAnalyze`

**Tab panels**: `planTab`, `analyzeTab`

**Map float buttons** (`mapXxx`):
`mapZoomIn`, `mapZoomOut`, `mapFit`, `mapLocate`

**Param panels** (`{mode}Params`):
`orthoParams`, `corridorParams`, `orbitParams`, `perimParams`

**Param inputs** (`{mode}{Param}`):
`orthoAlt`, `orthoSpeed`, `orthoFront`, `orthoSide`, `orthoPhoto`, `orthoGsd`
`corrAlt`, `corrSpeed`, `corrFront`, `corrSide`, `corrPhoto`
`orbitAlt`, `orbitSpeed`, `orbitRadius`, `orbitPhotos`, `orbitGimbal`, `orbitPasses`
`perimAlt`, `perimSpeed`, `perimOffset`, `perimSpacing`, `perimGimbal`, `perimDir`

**Slider display spans** (`{mode}{Param}Val`):
`orthoFrontVal`, `orthoSideVal`, `corrFrontVal`, `corrSideVal`, `orbitGimbalVal`, `perimGimbalVal`

**Stat cards**: `statArea`, `statPhotos`, `statDist`, `statTime`, `statWaypoints`, `statBattery`

**Advisories**: `gsdAdvisory`, `gsdAdvisoryVal`, `corrBearingAdvisory`, `corrBearingVal`

**Sidebar**: `sidebar`, `sidebarScroll`, `sidebarCloseBtn`, `sidebarOverlay`, `dragHandle`, `sidebarHeaderTitle`

**Map overlays**: `areaChip`, `areaChipVal`, `coordBar`, `coordLat`, `coordLng`

**Search**: `searchInput`, `searchResultsList`, `searchClearBtn`

**Analyze tab**: `srtDropZone`, `photoDropZone`, `deviationStats`, `devMax`, `devAvg`, `devRmse`, `coverageStats`, `covPhotos`, `covArea`, `mosaicCanvas`, `mosaicWrap`, `mosaicPlaceholder`

**Export buttons**: `exportKmz`, `exportCsv`, `exportKml`, `exportGeoJson`, `exportMosaic`

**File inputs**: `fileInKml`, `fileInSrt`, `fileInPhotos`

**Collapsible**: `sideloadHeader`, `sideloadBody`

**Canvas**: `altitudeProfile`, `mosaicCanvas`

**Toast area**: `toastArea`

## CDN Dependencies

```html
<!-- Fonts -->
fonts.googleapis.com — DM Sans (body) + DM Mono (mono)

<!-- CSS -->
leaflet@1.9.4/dist/leaflet.min.css

<!-- JS (sync, loaded before body) -->
leaflet@1.9.4/dist/leaflet.min.js
jszip@3.10.1/dist/jszip.min.js

<!-- JS (defer — guard with typeof Chart === 'undefined') -->
chart.js@4/dist/chart.umd.min.js
```

No npm, no bundler, no local assets.

## CSS Design System

**Background scale (dark theme):**
```css
--bg-0: #0a0b0f  /* page/app background */
--bg-1: #111318  /* toolbar, sidebar */
--bg-2: #1a1d24  /* cards, inputs */
--bg-3: #22252e  /* hover states */
--bg-4: #2a2d36  /* active/pressed */
```

**Accent colors:**
```css
--accent:  #4d9fff  /* blue — primary action, links */
--green:   #3ddc84  /* success, first waypoint */
--orange:  #ffab40  /* ortho/corridor path, warnings */
--red:     #ff5252  /* danger, last waypoint, errors */
--purple:  #e040fb  /* orbit/perimeter path */
```

**Each color has a `-dim` variant** (rgba at 0.15 alpha) for backgrounds.

**Typography:**
```css
--font-body: 'DM Sans'
--font-mono: 'DM Mono'
```

**Layout vars:**
```css
--sidebar-w: 340px  /* desktop; overridden at smaller breakpoints */
--toolbar-h: 52px
```

**Responsive breakpoints:**
- Phone: `<= 480px` — sidebar is bottom sheet (fixed, 65dvh max, `translateY`)
- Tablet: `481–768px` — sidebar is overlay panel on right (`translateX`)
- Laptop: `769–1024px` — sidebar is overlay panel, toolbar labels hidden
- Narrow desktop: `1025–1280px` — sidebar is inline, toolbar labels hidden
- Desktop: `>= 1025px` — sidebar is inline (pushes map with `margin-right`)

**Sidebar open/close mechanics:**
- Desktop (>= 1025px): `.sidebar.collapsed` hides it
- Below 1025px: `.sidebar.open` shows it; `#sidebarOverlay.visible` shows backdrop
- Phone: `.sidebar.open` slides up from bottom

**Param panels:** use `.param-panel` + `.visible` class. **Never** toggle `style.display`.

**Utility classes:** `.hidden` (display:none !important), `.sr-only` (screen-reader visually hidden), `.text-mono`, `.text-muted`, `.mt-8`, `.mb-8`, `.mt-12`, `.mb-12`

**Focus accessibility:** `:focus-visible` outline uses `--accent` at 2px. Mouse-click focus outline suppressed.

**Touch targets:** `@media (pointer: coarse)` sets min-height 44px on interactive elements.

**Z-index reference:**
- Toolbar: 900
- Map overlays (search, chips, float buttons): 400–500
- Toast area: 1000
- Sidebar (tablet/phone overlay): 500–600
- Sidebar backdrop: 490–590
- Rotation handle: zIndexOffset 1000 (Leaflet marker)
- Map overlays MUST use z-index >= 1000 to appear above Leaflet default layers

## Camera Specs (Hardcoded in `CAM` constant)

```js
'12mp': { sw: 9.65, sh: 7.24, f: 6.72, iw: 4032, ih: 3024 }
'48mp': { sw: 9.65, sh: 7.24, f: 6.72, iw: 8064, ih: 6048 }
```
`sw`/`sh` = sensor mm, `f` = focal mm, `iw`/`ih` = image pixels.

GSD formula: `gsd_cm = (sw * alt_m * 100) / (f * iw)`

## Working Rules

1. **Always copy before committing:** `cp "Flight Planner.html" index.html`
2. **Test all 4 modes** after any JS change: ortho, corridor, orbit, perimeter
3. **Plan object uses `.wps`** — never `.waypoints`
4. **Waypoint action is `'takePhoto'`** — never `'photo'`
5. **Mode strings:** `'ortho'`, `'corridor'`, `'orbit'`, `'perimeter'` — never `'perim'`
6. **Param panels:** toggle `.visible` class — never `style.display`
7. **Chart.js guard:** `if (typeof Chart === 'undefined') return` before any AltChart call
8. **iOS downloads:** use `navigator.share` via `Exporter.download()` — never create anchor directly
9. **Map overlays:** z-index >= 1000 to clear Leaflet tile layer stacking
10. **Sidebar state:** `AppState.reset()` inside `MapCtrl.clearAll()` fires all state events — listeners must be idempotent

## Pre-Push QA Checklist

1. All `getElementById()` / `querySelector()` IDs/selectors exist in HTML
2. All CSS classes used in JS have CSS rules (check `.visible`, `.active`, `.open`, `.collapsed`, `.drag-over`)
3. Event chain works end-to-end: draw polygon → `plan:regen` → `Planner.generate()` → `plan:generated` → `drawPlan` + `updateResults`
4. All toolbar buttons have click handlers (btnDrawArea, btnUploadKml, btnObstacles, btnLayers, btnClear, btnLocate, btnToggleSidebar)
5. All map float buttons have click handlers (mapZoomIn, mapZoomOut, mapFit, mapLocate)
6. All 4 param panels show/hide correctly on mode switch
7. Slider value displays update live
8. Exports produce valid files: KMZ unzips to `wpmz/waylines.wpml` (or split `waylines_N.wpml`), CSV has header row, KML has `<coordinates>`, GeoJSON has `type: "FeatureCollection"`
9. Responsive: toolbar scrolls horizontally on phone without overflow; sidebar opens/closes on all breakpoints
10. Search bar visible above map tile layers on all breakpoints
11. KMZ splits above 1999 waypoints (two `.wpml` files inside zip)
12. `typeof Chart === 'undefined'` guard prevents crash when Chart.js hasn't loaded
13. `index.html` is identical to `Flight Planner.html`

## Key Commands

```bash
# Local preview — just open in browser (no server needed)
open "Flight Planner.html"
# or double-click in Explorer

# Deploy workflow
cp "Flight Planner.html" index.html
git add "Flight Planner.html" index.html
git commit -m "describe change"
git push origin master
# GitHub Pages auto-deploys from master/index.html
```

## Known Patterns and Gotchas

- **No module system.** All cross-module calls are via globals on `window`. Order matters: `Bus` and `AppState` must be defined before all other sections.
- **Planner reads DOM inputs directly** inside `generate()`. If a param element doesn't exist, it falls back to a hardcoded default — no crash, but silent wrong value.
- **Corridor is ortho with auto-bearing.** `corridor()` calls `ortho()` internally with `gridAngle` overridden to the farthest-pair bearing.
- **Orbit uses `orbitCenter` from AppState**, not `siteCoords`. Drawing in orbit mode sets a single-click center point; polygon drawing is skipped.
- **Rotation handle exists only in ortho/corridor.** `MapCtrl.createRotationHandle()` no-ops if `siteCoords.length < 3`.
- **KMZ splits at `DJI_SPLIT = 1999`.** Files are named `waylines_0.wpml`, `waylines_1.wpml`, etc. when split. Single-file missions use `waylines.wpml`.
- **SRT parser expects DJI format**: `[latitude: N]`, `[longitude: N]`, `[rel_alt: N feets]` inside subtitle blocks. Non-DJI SRT files will parse as empty.
- **EXIF GPS parser** is a minimal `DataView` scanner. It reads only the first GPS IFD in APP1 Exif. Rational values (tag type 5) are decoded up to 3 components (DMS).
- **`debounceRegen` timer is 200ms.** Rapid param changes are coalesced. The regen bridge at the bottom of the file (`Bus.on('plan:regen')`) is the single point where `Planner.generate()` is called from the event bus.
- **`AppState.reset()` emits every state key.** Listeners on `state:siteCoords`, `state:flightPlan`, etc. will all fire on clear. This is intentional.
- **Chart.js is `defer`.** It may not be available when the first plan generates on page load if the user is very fast. The `typeof Chart === 'undefined'` guard in `AltChart.update` handles this silently.
- **iOS Safari file download** uses Web Share API (`navigator.share` + `navigator.canShare`). Falls back to anchor download on failure. Never call `_anchorDownload` directly — use `Exporter.download()`.
