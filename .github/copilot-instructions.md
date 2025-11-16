@ -0,0 +1,58 @@
## FoxholeReport – AI Coding Agent Guide

This repo builds a Leaflet-based Foxhole world map with routing, data overlays, and a custom tile/asset pipeline. Use these notes to work productively without guessing.

### Architecture
- Web app: TypeScript + Parcel in `src/` with Sass in `src/style/`. Entry is `src/index.html` → `src/main.ts`.
- Map + UI: Leaflet map setup and layer toggles in `src/main.ts`. Layers are registered via `Options` and `L.control.layers(...)`.
- Data/logic: Game API + control prediction in `src/API.ts` (fetches Foxhole endpoints per `config/shards.json`; kriging via `@sakitam-gis/kriging`). Regions listed in `src/Regions.ts`.
- Routing + overlays: Implemented in `src/IRouter.ts` (exports `Create`). Builds path-finders from `SimpleRoads.json`, renders roads/control via `VectorControlGrid`, labels via `TextGrid`.
- Workers: `src/PredictorWorker.ts`, `src/TileRenderWorker.ts` compiled to `workers/` (see scripts). Some assets loaded with `data-url:` imports (Parcel transformers).
- Service Worker: Registered in `src/main.ts` as module `./ServiceWorker.ts`.
- Asset pipeline: Node scripts in `scripts/` and .NET tools (`MapStitcher/`, `RoadSimplifier/`, `Tiler/`) generate stitched map and tiles.

### Build & Run
- Dev server: `npm run serve` (Parcel on port 1234) or `npm run watch` for watch mode.
- Full build: `npm run build` (builds workers and bundles site). To only compile Sass: `npm run build_sass`.
- Map/tile generation: `npm run map` relies on Node + .NET SDK. It runs:
  - `node scripts/roads.js` → writes `Roads.json`
  - `dotnet run --project RoadSimplifier ...` → `SimpleRoads.json`
  - `node scripts/export_roads_svg.js` → `Roads.svg`
  - `dotnet run --project MapStitcher ...` → `map.png`
  - `dotnet run --project Tiler ...` → `Tiles/{z}_{x}_{y}.webp`
- Cleanup/assets: `npm run clean` clears Parcel cache; `npm run assets` copies `Tiles` to `dist/Tiles`.

### Key Conventions & Patterns
- Sass modules: Each partial that uses variables must import them. In files like `src/style/_map.scss`, include `@use "variables" as *;` or use namespaced `variables.$white`. Do not rely on `style.scss` imports to leak into partials.
- Layers and bitmask: `src/main.ts` builds the overlay `Options` object and encodes active layers into a hash bitmask. If you add a layer, update both:
  - The `Options[...] = Router.<Layer>` map in `main.ts`
  - The switch statements in `overlayadd/overlayremove` and in `update_state` for bit positions and names (names are matched after stripping icon HTML).
- Router responsibilities (`src/IRouter.ts`):
  - Exposes `showX()/hideX()` to toggle icons/features via `VectorControlGrid`.
  - Constructs `PathFinder` graphs from `SimpleRoads.json` and provides alternative paths (Warden/Colonial vs shortest).
  - Adds labels with `TextGrid` and resource/base icons via `ControlLayer.addIcon(...)`.
- API data flow (`src/API.ts`):
  - Fetches war state and per-map `dynamic/public` items, normalizes to world coordinates using `remapXY`.
  - Trains a kriging variogram and predicts control for roads and towns (used in `IRouter.ts`).
- Data URLs & assets: Many UI images and fonts are inlined via `MapIcons.data_url(...)` and `data-url:` imports. Prefer this pattern for small assets used by controls.

### How-To Examples
- Add a new overlay (e.g., “Hospitals”):
  1) In `IRouter.ts`, create `Hospitals: L.layerGroup()` and `showHospitals()/hideHospitals()` to enable/disable relevant icons.
  2) In `main.ts`, add an entry in `Options[...] = Router.Hospitals` with an icon via `MapIcons.data_url(...)`.
  3) Extend the bitmask handling in `update_state`, `overlayadd`, and `overlayremove` with a unique bit and matching display name.

- Use Sass variables in a partial:
  - Top of file: `@use "variables" as *;` then use `$white`, `$grey-1`, etc. Alternatively: `@use "variables";` then `variables.$white`.

### Integration & Prereqs
- Node 18+ recommended. Parcel v2 configured via package.json; Sass transformer already included.
- .NET SDK (8+/9+) required for `MapStitcher`, `RoadSimplifier`, `Tiler` commands in `npm run map`.
- Runtime data: `config/shards.json` selects Foxhole API shard; towns in `config/towns.json`.

### Troubleshooting
- Missing Sass variables in partials: ensure the partial itself `@use`s `variables`.
- Layer not toggling or URL hash wrong: verify the layer display name matches switch/counter mapping in `main.ts` and that `Router.show*/hide*` mutate `VectorControlGrid` appropriately.
- Map generation fails on Windows: `.sh` scripts are optional; prefer `npm run map` which uses Node + .NET.

If anything here is unclear or you find a divergent pattern, leave a brief note in PR description so we can update this guide.