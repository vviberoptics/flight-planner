# Drone Flight Planner

**[Open the Planner](https://hfakes1.github.io/flight-planner/)**

Single-file browser tool for planning DJI Mini 4 Pro mapping missions. No install, no server, no account — just open and plan.

## Features

- **Three mission modes**: Ortho (nadir grid), Orbit (circular oblique), Perimeter (edge-offset oblique)
- **GSD calculator** with target advisory — know exactly what resolution you'll get
- **DJI Fly KMZ export** — per-waypoint heading, gimbal pitch, altitude
- **Live map** — satellite/street toggle, address search, GPS locate
- **Draggable vertices** — reshape your flight area on the map
- **Photo count + flight time** estimates with battery warnings
- **Zero dependencies** — runs entirely in the browser (Leaflet + JSZip via CDN)

## Mission Modes

### Ortho (Nadir Mapping)
Lawnmower grid pattern with camera pointing straight down (-90 gimbal). Set altitude, speed, overlap percentages, and photo resolution (12/48 MP). Produces orthomosaic-ready photo sets.

### Orbit (3D / Oblique)
Circular path around a point of interest. Configurable radius, gimbal pitch, photos per loop, and 1-3 altitude passes (100%/70%/40%). Each waypoint heading calculated to face center. Produces 3D models and site documentation.

### Perimeter (Edge Documentation)
Flies offset from polygon edges with camera aimed inward (or outward). Set offset distance, photo spacing, and gimbal pitch. Produces facade and edge documentation.

## GSD Reference (DJI Mini 4 Pro, 12 MP)

| Altitude (ft) | GSD (in/px) | Best For |
|---------------|-------------|----------|
| 100 | 0.72 | Inspection, crack detail |
| 150 | 1.08 | Topo survey, earthwork |
| 200 | 1.44 | Construction progress |
| 300 | 2.16 | Large area overview |

## Overlap Guide

| Terrain | Front | Side |
|---------|-------|------|
| Flat / open | 70% | 65% |
| Mixed / suburban | 75% | 70% |
| Dense / complex | 80% | 75% |
| Vegetation / trees | 85% | 80% |

## Export Formats

| Format | Use |
|--------|-----|
| DJI Fly KMZ | Primary — load directly into DJI Fly waypoint missions |
| CSV | Waypoint list for external tools |
| KML | Flight path visualization in Google Earth |
| Litchi CSV | Litchi app format (legacy) |

## iPhone + RC-N2 Workflow

1. Plan your mission and export **DJI Fly KMZ**
2. In DJI Fly: Waypoints > save 2 dummy points > exit
3. Files app > On My iPhone > DJI Fly > wayline_mission > replace newest `.kmz` with yours
4. Reopen DJI Fly > Waypoints > open mission > fly

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| D | Draw area |
| L | GPS locate |
| F | Fit to bounds |
| M | Toggle map layer |
| Esc | Cancel draw |
| Del | Clear all |
| +/- | Zoom in/out |

## Camera Specs (Hardcoded)

DJI Mini 4 Pro: sensor 9.65x7.24mm, focal 6.72mm, 4032x3024 (12MP) / 8064x6048 (48MP)

## License

MIT — see [LICENSE](LICENSE)
