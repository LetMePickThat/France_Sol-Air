# France Sol-Air — CLAUDE.md

## Project Overview

**France Sol-Air** is a browser-based GBAD (Ground Based Air Defense) deployment planning and radar coverage analysis tool. It lets military planners place radars, missile launchers, and C2 (command-and-control) nodes on an interactive map and compute radar intervisibility/coverage using real terrain elevation data.

The application is bilingual (French default, English toggle) and complies with DSFR (Système de Design de la République Française) web standards.

## Architecture

- **Single-file SPA**: `radar_v5.html` (~2,750 lines) — all HTML, CSS, and JavaScript in one file
- **Standalone help page**: `aide.html` — separate DSFR-styled page, bilingual (FR/EN toggle)
- **No build system**: open the file directly in a browser or serve it statically
- **No backend**: all computation is client-side
- **`index.html`**: minimal redirect page to `radar_v5.html` (for GitHub Pages)

## File Structure

```
France_Sol-Air/
├── radar_v5.html       # Main application (single-file SPA)
├── aide.html           # Standalone help page (FR/EN)
├── index.html          # GitHub Pages redirect
└── images/
    ├── favicon.svg         # DSFR-compliant SVG favicon
    └── Aide_image_IHM.png  # Interface screenshot used in aide.html
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Mapping | Leaflet 1.9.4 (CDN) |
| Elevation tiles | AWS Terrarium S3 (RGB-encoded PNG, ~30 m/px at zoom 11) |
| Base maps | OpenStreetMap / CARTO Light / CARTO Dark |
| Fonts | Marianne (DSFR, CDN) |
| Language | Vanilla ES6+ JavaScript, HTML5, CSS3 |
| Export/Import | KML 2.2 with custom `spm:` namespace |

## Key Features

- **Radar coverage computation**: raycast line-of-sight over terrain with refraction model (k-factor: 4/3, 1.0, 1.15, 1.6), results rendered as canvas overlay
- **DEM quality toggle**: Rapide (zoom ≤ 11, ~52 m) / Haute Précision (zoom ≤ 12, ~26 m) in CARTE tab, with live metric resolution estimate
- **GBAD entities**: Radars (blue circles), Launchers (orange triangles), C2 Nodes (blue squares) — all draggable, fully configurable
- **C2 hierarchy**: parent/subordinate relationships, distance-constraint circles, LOS link validation (fiber vs radio)
- **Local summit search**: draw a polygon, find terrain peaks by elevation threshold
- **KML import/export**: full scenario round-trip with all parameters in `spm:` ExtendedData
- **Multi-altitude coverage**: user-defined target altitudes (default: 0, 5, 40, 500 m)
- **Coordinate tools**: DMS ↔ decimal degrees conversion
- **i18n**: 70+ string pairs, function-based interpolation, FR/EN toggle
- **Theme toggle**: light/dark per DSFR spec
- **Compact UI mode**: `@media (max-height: 820px)` reduces chrome to preserve form space on low-res screens
- **Help page**: `aide.html` with 14 sections, sidebar scroll-spy, FR/EN toggle, DSFR styling

## Running the App

```bash
# Simplest — open directly:
start radar_v5.html

# Or serve locally (avoids CORS issues with some browsers):
python -m http.server 8000
# then visit http://localhost:8000/radar_v5.html
```

No `npm install`, no build step, no backend needed. Internet access required for CDN assets and AWS elevation tiles.

## Code Organization (inside `radar_v5.html`)

The file is structured as a single HTML document with embedded `<style>` and `<script>` blocks:

- **Globals / constants**: map config, tile cache, entity stores (`radars`, `launchers`, `c2nodes`), `demZoomCap` (DEM quality toggle)
- **i18n**: `translations` object + `t()` helper; keys include `calcSection`, `calcFast`, `calcHD`, `demResEstimate`
- **Geospatial math**: `haversine`, `latlngToTile`, `bilinearElevation`, `checkLOSAsync`, `buildDEM`
- **Entity management**: add/remove/update functions per entity type, C2 hierarchy cycle detection
- **UI**: tab panels, accordion sections, entity forms, map tooltip, result badge
- **Coverage engine**: `computeCoverage()` — fetches tiles, raycasts, renders canvas overlay
- **DEM quality**: `updateDemResEstimate()` — estimates resolution in metres for current deployment centroid
- **KML**: `exportKML()` / `importKML()` with full SPM namespace serialization
- **Summit search**: `startPeakSearch()`, `findPeaksInPolygon()`

## Important Technical Details

- Elevation tiles are fetched with `crossOrigin = 'anonymous'`; tile URL pattern: `https://elevation-tiles-prod.s3.amazonaws.com/terrarium/{z}/{x}/{y}.png`
- Tile cache is a `Map<string, Promise<HTMLImageElement>>` shared across all async callers
- `SPOT_ZOOM = 11` (hardcoded) — used for LOS point checks and summit search (~52 m in France)
- `demZoomCap` — controls coverage DEM resolution: 11 (Rapide) or 12 (Haute Précision). Coverage target is 30 m/px, auto-selected zoom capped by `demZoomCap`
- Coverage DEM zoom formula: `zoom = round(log2(R_MERC * 2π * cos(lat) / (256 * 30)))`, clamped to `[8, demZoomCap]`
- Coverage rays are computed asynchronously with `setTimeout(0)` yields every 100 rays to keep the UI responsive
- KML export uses SVG balloon content to render azimuth/elevation sector diagrams inline
- Cycle detection in C2 hierarchy uses iterative parent-chain traversal to prevent infinite loops
- Range rings "Tous" checkbox is checked by default at startup

## Help Page (`aide.html`)

Standalone DSFR-styled page with sidebar navigation and scroll-spy via `IntersectionObserver`.

**Sections**: Introduction · Interface · Démarrage rapide · Unités (Radars, Lanceurs, C2) · Cible & Atmo · Carte · Sommets · Outils · KML · Couverture radar · Glossaire (22 entries)

**Bilingual**: FR/EN toggle (`.seg #help-lang-toggle`) switches visibility via `data-help-lang` attribute on `<html>`. FR content in `.help-fr` divs, EN in `.help-en`. Sidebar nav is duplicated for each language.

**Favicon**: `images/favicon.svg` — DSFR-compliant: blue `#000091` background, 6 px tricolor strip (blue/white/red) on left edge per DSFR service identity rules, white radar symbol (two arcs + sweep line).

## Deployment

Static files only. Currently hosted on GitHub Pages (`LetMePickThat/France_Sol-Air`). To deploy: push all files to any static host.
