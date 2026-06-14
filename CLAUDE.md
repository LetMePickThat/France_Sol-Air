# France Sol-Air — CLAUDE.md

## Project Overview

**France Sol-Air** is a browser-based GBAD (Ground Based Air Defense) deployment planning and radar coverage analysis tool. It lets military planners place radars, missile launchers, and C2 (command-and-control) nodes on an interactive map and compute radar intervisibility/coverage using real terrain elevation data.

The application is bilingual (French default, English toggle) and complies with DSFR (Système de Design de la République Française) web standards.

## Architecture

- **Single-file SPA**: `radar_v6.html` (~6,150 lines) — all HTML, CSS, and JavaScript in one file; ENGAGEMENT tab is fully visible
- **Standalone help page**: `aide.html` — separate DSFR-styled page, bilingual (FR/EN toggle)
- **No build system**: open the file directly in a browser or serve it statically
- **No backend**: all computation is client-side
- **`index.html`**: minimal redirect page to `radar_v6.html` (for GitHub Pages)

## File Structure

```
France_Sol-Air/
├── radar_v6.html           # Main application (single-file SPA) — all 4 tabs enabled
├── aide.html               # Standalone help page (FR/EN)
├── index.html              # GitHub Pages redirect → radar_v6.html
├── missiles/
│   └── generic_mrm.json    # External missile config (medium range, ~10 km)
└── images/
    ├── favicon.svg             # DSFR-compliant SVG favicon
    └── Aide_image_IHM.png      # Interface screenshot used in aide.html
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
- **i18n**: 80+ string pairs, function-based interpolation, FR/EN toggle
- **LocalStorage save system**: 3 named slots + auto-save with F5 protection (`beforeunload`); slot buttons in OUTILS accordion; `showSaveFilePicker()` / `prompt()` fallback for KML and PNG exports
- **Theme toggle**: light/dark per DSFR spec
- **Compact UI mode**: `@media (max-height: 820px)` reduces chrome to preserve form space on low-res screens
- **Help page**: `aide.html` with 14 sections, sidebar scroll-spy, FR/EN toggle, DSFR styling
- **Threat simulation**: waypoint-based target trajectory, configurable speed/altitude per segment, simulation player with speed control
- **Kinematic reachability (Phase 1)**: PIP engine, engagement windows, 4 event instants per (radar, launcher) pair — see engagement section below

## Running the App

```bash
# Simplest — open directly:
start radar_v6.html

# Or serve locally (avoids CORS issues with some browsers):
python -m http.server 8000
# then visit http://localhost:8000/radar_v6.html
```

No `npm install`, no build step, no backend needed. Internet access required for CDN assets and AWS elevation tiles.

## Code Organization (inside `radar_v6.html`)

The file is structured as a single HTML document with embedded `<style>` and `<script>` blocks:

- **Globals / constants**: map config, tile cache, entity stores (`radars`, `launchers`, `c2nodes`), `demZoomCap` (DEM quality toggle)
- **i18n**: `translations` object + `t()` helper
- **Geospatial math**: `haversine`, `latlngToTile`, `bilinearElevation`, `checkLOSAsync`, `buildDEM`
- **Entity management**: add/remove/update functions per entity type, C2 hierarchy cycle detection
- **UI**: tab panels, accordion sections, entity forms, map tooltip, result badge
- **Coverage engine**: `computeCoverage()` — fetches tiles, raycasts, renders canvas overlay
- **DEM quality**: `updateDemResEstimate()` — estimates resolution in metres for current deployment centroid
- **KML**: `exportKML()` / `importKML()` with full SPM namespace serialization
- **Summit search**: `startPeakSearch()`, `findPeaksInPolygon()`
- **Threat simulation**: `simPlay()`, `simTick()`, `interpTargetPosAt()`, waypoint editor, progress bar
- **Missile kinematics**: `missileTimeToReach()`, `interpMissileV()`, `BUNDLED_MISSILE_CONFIGS`, velocity-profile integration
- **PIP engine**: `solvePIP()`, `solvePIPScan()`, `solvePIPLive()` — see engagement section
- **Engagement windows**: `computeEngWinForLauncher()`, `computeAllEngWindows()` — async, version-guarded
- **Real-time display**: `updateRealtimePIPs()`, `renderPIPOverlay()`, `renderProgressGradient()`

## Engagement / Kinematic Reachability Engine (Phase 1)

Implements spec "FSA — Spécification : Atteignabilité cinétique & fenêtre d'engagement".

### Four event instants per (radar, launcher) pair

| Instant | Meaning |
|---------|---------|
| `t_det` | First detection sample |
| `t_tir_min` | `fo.t` — earliest fire authorisation (= t_det + C2 chain delay) |
| `t_tir_first` | First feasible t_tir ≥ t_tir_min (kinematic check passes) |
| `t_tir_last` | Last feasible t_tir; `close_reason`: `envelope_exhausted` or `data_exhausted` |

### PIP solver variants

Three variants, all using the same scan + bisect approach on `g(T) = (T − T_avail) − tof`:

| Function | Velocity reference | dt | Use |
|----------|-------------------|----|-----|
| `solvePIP(launcher, cfg, T_fire)` | Constant velocity estimated at `T_avail` (look-back 3s) | 0.05 s | Fire order result, missile animation target |
| `solvePIPScan(launcher, cfg, T_fire)` | Oracle `interpTargetPosAt(T)` | 0.5 s | Engagement window sweep — planning tool only |
| `solvePIPLive(launcher, cfg, T_fire, tCurrent)` | Constant velocity estimated at `tCurrent` (look-back 3s) | 0.5 s | Per-tick live PIP update during simulation |

**Tactical vs oracle split (spec §4):**
- `solvePIP` / `solvePIPLive` = "chemin rapide" — constant-velocity extrapolation from observed track; models what the real ground system computes without knowledge of future maneuvers
- `solvePIPScan` = "chemin complet" — oracle; used only for the engagement window planner where full trajectory knowledge is acceptable

### Live PIP tracking

`updateRealtimePIPs(tCurrent)` calls `solvePIPLive` on every simulation tick once a missile is in flight. The PIP diamond marker, launcher→PIP line, and missile position all update in real time as target maneuvers become visible on the radar track.

### Missile configs

Missile configs are defined in `BUNDLED_MISSILE_CONFIGS` (inline JS) and optionally in external JSON files under `missiles/`. Format: `{ id, label, kinematics: { launchDelay_s, boostEnd_s, maxFlightTime_s, velocityProfile[] }, envelope: { slantRangeMin_m, slantRangeMax_m, altMin_m, altMax_m, vFloor_ms, tofMax_s }, warhead: { lethalRadius_m } }`.

Current bundled configs:
- `generic_mrm` — MSAM portée moyenne (~10 km, tofMax 20 s)
- `generic_mk1` — MSAM longue portée notionnel (~35 km, tofMax 120 s)

### State variables

```javascript
let _engWindows      = [];   // parallel to _fireOrders
let _engWindowsReady = false;
let _detVersion      = 0;    // incremented on scenario invalidation; async loops abort on mismatch
```

### Engagement tab visibility

The ENGAGEMENT tab is fully enabled in `radar_v6.html` (tab button visible, no `display:none`). The dev proto `radarV6_eng_proto.html` was deleted on 2026-06-14; `radar_v6.html` is the sole working file.

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
- `runDetectionAnalysis(onProgress, doEngWindows)` — second arg `true` only when called from `runSequenceCalc`; prevents slow window computation during `simPlay`'s per-tick silent re-runs

## Help Page (`aide.html`)

Standalone DSFR-styled page with sidebar navigation and scroll-spy via `IntersectionObserver`.

**Sections**: Introduction · Interface · Démarrage rapide · Unités (Radars, Lanceurs, C2) · Cible & Atmo · Carte · Sommets · Outils · KML · Couverture radar · Glossaire (22 entries)

**Bilingual**: FR/EN toggle (`.seg #help-lang-toggle`) switches visibility via `data-help-lang` attribute on `<html>`. FR content in `.help-fr` divs, EN in `.help-en`. Sidebar nav is duplicated for each language.

**Favicon**: `images/favicon.svg` — DSFR-compliant: blue `#000091` background, 6 px tricolor strip (blue/white/red) on left edge per DSFR service identity rules, white radar symbol (two arcs + sweep line).

## Deployment

Static files only. Currently hosted on GitHub Pages (`LetMePickThat/France_Sol-Air`). To deploy: push all files to any static host.
