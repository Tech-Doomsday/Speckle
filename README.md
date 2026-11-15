# Speckle

Speckle turns NISAR radar data into evidence of how the ground is changing. It reads NISAR GUNW and GCOV products from NASA Earthdata, fetching only the bytes it needs, and runs thirteen preprocessing steps, all visible in the interface. Users explore a 3D map with a timeline of ascending and descending passes.

Each calamity has its own independent pipeline. Seventeen are designed and four are live: earthquake, tsunami, flood and deforestation. The earthquake pipeline compares measured ground movement with a physical fault model (Okada) and can flag possible quakes missing from catalogues. Tsunami evidence comes from ocean buoys and a wave simulation, so a linked earthquake–tsunami event rests on two independent data types. Flood and deforestation pipelines track radar backscatter changes to show spreading water and forest loss. Results are ranked by strength of evidence, and the app says plainly when data is insufficient.

Planned work: tracing slight ground dents that reveal hidden collapse, river erosion effects on soil and vegetation, separating human-driven from natural forest decline, and radar changes before events.

The code is public on GitHub under an open licence. Its clear SAR insights make it a strong tool for data scientists and for planners.

---

# JUST COPY AND PASTE THIS COMMAND
# OR DOWNLOAD JUST `run.bat` AND RUN IT IN YOUR TERMINAL

### Copy and paste into PowerShell (or Command Prompt), press Enter:

```
curl.exe -L -o run.bat https://raw.githubusercontent.com/Tech-Doomsday/Speckle/main/run.bat
.\run.bat
```

### Or, if you downloaded [`run.bat`](https://raw.githubusercontent.com/Tech-Doomsday/Speckle/main/run.bat) (right-click → *Save link as*), open a terminal and run:

```
cd Downloads
.\run.bat
```

(Or just double-click it. Run it from whichever folder you saved it in.)

**Windows only. Needs [Git](https://git-scm.com/download/win), [Python 3.11+](https://www.python.org/downloads/) and [Node 20+](https://nodejs.org/) installed. Nothing else, no setup.**

## What `run.bat` does for you

1. Checks that Git, Python and Node are installed.
2. Clones this repo into a `Speckle` folder next to the file (skipped if it is already inside the repo).
3. Creates a Python environment and installs everything (first run only).
4. Sets up the credentials file.
5. Opens **two windows**: the engine (port 8900) and the website (port 3000).
6. Opens **http://localhost:3000** in your browser when it is ready. The first run takes a minute or two.

To stop Speckle, close the two windows (**Speckle Engine** and **Speckle UI**). To start it again, run `run.bat` again.

---

## Project layout

Global SAR hazard detection engine — **NISAR** primary, **Sentinel-1** fallback. One Python engine
(retrieval by HDF5 range reads → P01–P13 preprocessing → routed hazard pipelines → verdicts), one
Next.js website (cinematic landing + console + globe map), and a debug hub for watching every step.
Run the engine locally, or on a **Google Colab** runtime and use its fast datacenter link to NASA/ASF.

```
engine/          FastAPI app (engine/api) + retrieval + common chain + hazard pipelines
app-ui/          the website: landing film, console, map, background jobs, runs, verdicts
colab/           Colab drivers: notebook, scan script, retrieval hub, zip builders
tools/debug_hub  local debug layer: retrieval / health / processing / pipelines in one page
config/ supabase/ tests/ docs/
```

---

## 1. Manual installation (without `run.bat`)

Two terminals: **terminal 1** is the engine API, **terminal 2** is the website. Python **≥ 3.11**
and Node.js 20+ are needed. Requirements are unified — one `requirements.txt` covers the serving
layer, retrieval and the common chain.

### 1.1 Credentials (once)

```bash
cp .env.example .env
```

Then edit `.env` and set your NASA Earthdata token: generate it at
<https://urs.earthdata.nasa.gov/profile> → **Generate Token**, and authorise
**"Alaska Satellite Facility Data Access"** in the same profile.

```
EARTHDATA_TOKEN=...
```

Everything else in `.env` is optional (data dir, Supabase verdict cache, scan opt-in, CORS origins,
timeouts). `NISAR_ENABLE_SCANS=1` is required before the UI's *Scan* control is enabled.
The map loads its SAR pictures as soon as a place is pinned (`NEXT_PUBLIC_AUTOLOAD_DIFFS=0` turns that off). There is no
access key: the engine serves public NISAR data with your Earthdata credentials, so keep it on your own machine, or put
your own gate in front of it before exposing it through a tunnel or a public host.

**Ascending + descending.** One SAR geometry only sees part of the ground, so the earthquake verdict also merges the best
ascending and best descending pair (`engine/pipelines/earthquake_coseismic/merge.py`; settings `merge_geometries`,
`merge_decompose`, `merge_min_geometry_det` in `config/common.yaml`). The *See why* player has a Merged / Ascending /
Descending switch (Merged is the default). Detection is merged; east/up movement is solved only where both passes saw the
pixel and the event window is known (north-south motion assumed zero); line-of-sight values are never averaged. The
verdict status and forward-model screens still use the single best pair.

### 1.2 Terminal 1 — the engine API (port 8900)

From the repo root:

```bash
pip install -r requirements.txt
python -m uvicorn engine.api.main:app --port 8900
```

- API root: <http://127.0.0.1:8900> — interactive docs at <http://127.0.0.1:8900/docs>
- The two `localhost:3000` dev origins are always allowed as CORS origins.

### 1.3 Terminal 2 — the website (port 3000)

```bash
cd app-ui
npm install
npm run dev
```

Open <http://localhost:3000> — the cinematic landing. Its **Start exploring** button (and the nav
**Console** button) enter the console at `/console`, which links into the map, background jobs,
runs and verdicts. Useful scripts inside `app-ui/`:

```bash
npm run typecheck          # tsc --noEmit
npm run guard:no-science   # UI must not compute science, only the engine does
npm run build              # production build
npm run test:e2e           # Playwright
```

The UI defaults to the engine at `http://127.0.0.1:8900` (`NEXT_PUBLIC_ENGINE_API` in
`app-ui/next.config.ts`). To point it at a remote engine (e.g. Colab) click the **engine pill** in
the nav bar (top-left, next to Landing · Map · Background), paste the URL, press **Link** — it is
health-checked and remembered in the browser. *Use default* goes back to the local engine.

### 1.4 Running the backend on Colab instead

Same engine, Colab's network. Build the package on your machine, run it in Colab, and link your
local UI to it.

**On your machine (repo root):**

```bash
python colab/build_colab_zip.py     # -> dist/speckle-colab.zip (no node_modules, no data/)
```

The zip does **not** contain your `.env` by default. Add `EARTHDATA_TOKEN` under
**Colab Secrets** (key icon, enable notebook access) — or answer the hidden prompt in the notebook.
(`--include-env` restores the old pack-the-token behaviour; rotate the credentials if that zip
leaves your machine.)

**In Colab** (CPU runtime; no engine code uses a GPU):

1. Upload the zip to `/content` (left file panel) and open `colab/Speckle_Colab.ipynb`
   (File → Upload notebook).
2. Run **cell 1** — unzip + `.env` + dependencies + credentials + machine tuning. If it says
   "Restart session": Runtime → Restart session, run the cell again (it skips the install).
3. Run the scan cells (2: `nc.scan(LON, LAT, ...)` — live progress, granule N/M, MB on the wire;
   3: `nc.show(state)` results table + evidence layers; 4: `nc.export(state)` + download zip).

Prefer a plain script (its own setup, output streams into the cell):

```bash
!python -u /content/speckle/colab/scan.py --lon 90.41849 --lat 23.748 --lookback 60
```

**Expose the engine API to your local UI** — run in a Colab cell:

```python
url = nc.serve(8900, tunnel=True)       # prints NEXT_PUBLIC_ENGINE_API=https://...trycloudflare.com
# nc.stop_serve()
```

or as a shell cell:

```bash
!python -u /content/speckle/colab/serve.py
```

Copy the printed `https://...trycloudflare.com` URL, then on your machine either click the engine
pill in the UI nav bar → paste → **Link**, or put `NEXT_PUBLIC_ENGINE_API=<that url>` in
`app-ui/.env.local` and restart `npm run dev`.

> **The tunnel URL has no authentication and scans are enabled: anyone holding it can spend your
> Earthdata token.** Keep it private and stop the cell when done.

---

## 2. The debug hub in Colab — data retrieval + your own data science

`nisar_hub.pyz` is a one-file retrieval hub: type a place and a size, see exactly which 10 km engine
tiles will be downloaded, download them, then browse **every stored layer as an image**. The point:
retrieval runs in Colab (fast link to ASF), while the data analysis and data science stay yours —
every clip is a real array on disk you can open in notebook cells.

**Build it (repo root):**

```bash
python colab/build_hub.py            # -> dist/nisar_hub.pyz (~0.5 MB: engine + config + the page)
```

| Option | Effect |
| --- | --- |
| `--no-env` | no credentials inside: Colab reads `EARTHDATA_TOKEN` from Secrets or prompts (use for a file you hand to others) |
| `--wheels` | also packs pinned dependency wheels (~90 MB); Colab installs from them first |
| `--out PATH`, `--py 3.11` | output path; Python version for the wheels (default 3.12) |

**Run it in Colab** — upload `nisar_hub.pyz` to `/content`, then one notebook cell:

```python
import runpy; runpy.run_path("/content/nisar_hub.pyz", run_name="__main__")
```

(or a shell cell: `!EARTHDATA_TOKEN=... python -u /content/nisar_hub.pyz`). It unpacks to
`/content/nisar-hub`, installs what Colab lacks (~a minute, once per runtime), opens a Cloudflare
tunnel and prints **one link ending in `?k=<secret>`**. The tunnel URL is public — the secret is
required on every request (the link sets it as a cookie and strips it from the address bar).
**Treat the link like a password.**

**What you get on the page:**

- **Retrieve** — type latitude, longitude and a size in km; the page shows the 10 km engine tiles
  that cover it (a 30 km box is 9–16 whole tiles) and which are already on disk. *Plan: measure
  size* finds the granules and the exact bytes **without downloading**; *Download* fetches them
  tile by tile. The queue runs inside Colab, so closing the tab does not stop it.
- **Datasets** — disk use, a map of every stored tile, per retrieval its granules, and every stored
  array as an image: click one for measured min / max / mean / histogram, click a pixel to read its
  stored value. *Download zip* keeps a retrieval.

Everything is stored under **`/content/datasets`** (local disk — SQLite needs real file locks, so
not Drive) and is **wiped when the session ends**: use *Download zip* or `nc.to_drive(...)`.
Stop the hub with `import nisar_colab as nc; nc.stop_serve()`. Options via
`os.environ["NISAR_HUB_ARGS"] = "--port 8901 --concurrency 32"` before the cell (`--data-dir`,
`--no-tunnel` also exist); at most 25 tiles per request (`NISAR_HUB_MAX_TILES`), because a 10 km
tile of GCOV + GUNW is hundreds of MB.

### Screenshots — using the hub in Colab

<!-- GALLERY:docs/pics/colab/debug_hub -->
_No screenshots in `docs/pics/colab/debug_hub/` yet._

<!-- /GALLERY -->

---

## 3. The debug hub on desktop

`tools/debug_hub` is the local debug layer over the real engine path — four tabs, one page. It runs
the *actual* code (`engine.retrieval NisarClient` → `engine.common` P01–P13 → `SceneBundle` →
`Registry.plan` → featured pipelines) through a recording proxy: observe only, **no mocks in live
mode** — a missing dependency or ASF being down shows the engine's own error text.

```bash
python tools/hub.py --open            # http://127.0.0.1:8765  (loopback only)
python tools/hub.py --allow-selftest  # also offers the synthetic SELF-TEST source (tests the hub itself)
```

(`python -m debug_hub` from `tools/` is the same thing; flags: `--port`, `--open`,
`--allow-selftest`.)

Type a latitude / longitude — or paste `34.05, -118.25` into either box; the tile (MGRS id, EPSG,
bbox) updates as you type. **RUN** drives a full engine pass and streams what happens:

| Tab | Shows |
| --- | --- |
| **DATA RETRIEVAL** | the request; live transfer counters (requests, wire vs useful bytes, retries, cache tiers); every retriever call with timing; granules found + footprints; every clip as raw HDF5 layers — image, shape, dtype, stats, hover pixel value + lon/lat, click to enlarge |
| **HEALTH** | environment, credentials (presence only), dependencies found by scanning imports, module import test, storage / ancillary files, stub-vs-implemented per stage, pipeline file state, the data-needs matrix; on demand: network reachability, pytest |
| **DATA PROCESSING** | input = the retrieval output; P01–P13 one by one (reads, writes, flags, assumptions, arrays); the SceneBundle; the hand-off table — for each declared pipeline, what `Registry.plan` would do and which declared inputs are present |
| **PIPELINES** | input + output of the featured pipelines — **earthquake (3.01)** and **hurricane (3.13 tropical_cyclone)** — click a card to expand: routing decision + reasons, the measurement pair, status, narrative, raw_p (a rank score, never a probability), evidence items with support numbers, evidence layers as images, provenance, warnings |

SELF-TEST is off by default, stamps a red banner on every screen and is never mixed with live data —
its scene is the repo's calibrated earthquake fixture (the T-8 truth bowl), with the injection
labelled in the UI. Security: the server binds `127.0.0.1` only, rejects foreign `Host` headers,
requires a per-process token on every POST and runs one job at a time.

### Screenshots — using the hub on desktop

<!-- GALLERY:docs/pics/debug_hub -->
_No screenshots in `docs/pics/debug_hub/` yet._

<!-- /GALLERY -->

---

## 4. The website

| Route | What it is |
| --- | --- |
| `/` | the Speckle cinematic landing — a scroll-driven film over a WebGL Earth telling the mission story |
| `/console` | the plain-language console home: engine status strip, the steps, glossary |
| `/map` | the globe: search a place, see granule footprints and timelines, findings, scans |
| `/background/retrieval` | background data retrieval — "Plan & download" writes retrievals (manifest + stored clips) under the engine's data directory |
| `/background/overview`, `/background/pipelines`, `/background/preprocessing`, `/background/health` | background job boards for the engine's stages |
| `/runs`, `/runs/[id]` | run records |
| `/verdicts`, `/verdicts/[bundle]` | verdicts per bundle |
| engine API `:8900` | FastAPI — everything the pages show comes from here |

The engine pill in the nav bar switches between the local engine and a linked remote engine (e.g.
the Colab tunnel from §1.4).

### The SAR overlay — how to get good insight

The overlay puts real SAR pixels (and pipeline evidence layers) on top of the map, georeferenced by
the engine:

1. **Pick a granule.** On `/map`, search or click a footprint and open its detail panel; the
   **Layers** tab lists the granule's stored layers — the engine decides which are previewable.
2. **Show it on the map.** Press **"Show on map"** on a layer (or use the SAR overlay control):
   choose the layer, and the image is drawn inside the granule's footprint using the engine's
   `corners_lonlat`. If the engine returns no corners, nothing is drawn and the engine's reason is
   shown instead.
3. **Tune the reading.** The control exposes:
   - **scale** — `auto`, `linear`, `dB`. For backscatter contrast (flood scars, liquefaction,
     deforestation edges) `dB` usually separates far more than `linear`;
   - **opacity** — drop it (~0.4–0.6) to blend SAR with the base map and see *where* a bright/dark
     feature sits on the ground; keep it high to read texture;
   - **validate** — re-derive the preview against the engine's quality rules when a layer looks odd;
   - the **legend** names the colour map and unit; **hover** reads a pixel's value + lon/lat,
     **click** enlarges.
4. **Read evidence, not just imagery.** Pipeline findings overlay their evidence layers on the map
   with per-layer opacity (the map keeps a small LRU of layers) — that is where "why did the engine
   fire here" becomes visible.
5. **Compare in time.** Use the granule timeline to find pre/post-event acquisitions and the
   explorer's before/after view for change pairs — the fastest way to see what an event actually
   moved.

### Screenshots — the website and its overlays

<!-- GALLERY:docs/pics/overlay_usage -->
_No screenshots in `docs/pics/overlay_usage/` yet._

<!-- /GALLERY -->

---

## 5. Screenshot galleries

Each gallery above is a folder under `docs/pics/`. Drop screenshots in (`.png`, `.jpg`, `.jpeg`,
`.gif`, `.webp`, `.svg`), name them `01_...`, `02_...` to control the order — the caption is the
file name without the number prefix and extension — then regenerate the galleries:

```bash
python dev_scripts/readme_gallery.py            # rewrite the gallery blocks in README.md
python dev_scripts/readme_gallery.py --check    # exit 1 if README.md is out of date (CI)
```

Nothing outside the `<!-- GALLERY:... -->` markers is touched.

## 6. Tests

```bash
python -m pytest -q                                   # the maintained suite (tests + pipelines + lib)
EQ_LIVE=1 python -m pytest -m live -q                 # opt-in live-network tests
```

## 7. The Earthquake observatory — its own page at `/earthquakes`

The **Earthquake observatory** is a separate page of the app (`/earthquakes`), a
guided investigation room over the same engine API the console and map use. Nothing about the
engine, the console, the map or the background pages changed — this is a purely additive
front-end feature that talks to the same FastAPI surface (§5.2 tile-at, §5.5 granules, §5.3
scans/runs, canonical verdicts).

**Open it**: scroll the landing and use the **Explore earthquakes** button on the earthquake
sector card, or go straight to `http://localhost:3000/earthquakes`. It does not replace anything
on the landing: the nav **Go and play** button and the finale **Start exploring** button both lead
to the Map (`/map`).

**The eight chapters** — Event (live USGS ComCat catalogue via the engine + the repo's curated
disaster list), Coverage (the tile's NISAR GUNW/GCOV inventory), Scan (the real run workflow,
progress straight from the engine's heartbeat), Trace (the LOS displacement profile from the
engine's own raster), Verdict (the one-sentence answer with tier, probability and evidence),
Map (MGRS tile + epicentre on Leaflet), Evidence (every claim, source and skipped input), and
Archive (the engine's stored verdicts per tile — the same `verdicts.db` the console reads).

**Where it lives**:

| Path | What |
|---|---|
| `app-ui/src/components/quake/` | the observatory UI (sections, charts, globe, advanced panel) |
| `app-ui/src/lib/engine/` | its typed engine client — transport only, zero science (the repo's `src/lib` rule) |
| `app-ui/src/components/quake/engineHooks.ts` | the TanStack Query hooks over that client |
| `app-ui/src/components/quake/data.ts` + `utils.ts` | the engine-parameter labels module + the `cn()` helper |
| `app-ui/src/app/(landing)/page.tsx` | the cinematic landing film (nothing else) |
| `app-ui/src/app/(landing)/earthquakes/page.tsx` | the `/earthquakes` route that mounts the observatory |
| `app-ui/src/app/providers.tsx` | the client-side query provider for the view |
| `app-ui/src/app/api/engine-keepalive/route.ts` | dev-only convenience: GET it to verify/start the engine |

New UI dependencies in `app-ui/`: `@tanstack/react-query`, `recharts`, `clsx`,
`tailwind-merge` (run `npm install` once after pulling). The observatory honours the same
engine conventions as the rest of the UI: it defaults to `http://127.0.0.1:8900` on localhost,
and otherwise follows `NEXT_PUBLIC_ENGINE_API` — nothing new to configure if you already run the
pair. With no Earthdata credentials configured the observatory shows the engine's own honest
reasons verbatim (set `EARTHDATA_TOKEN` per §1.1 to unlock granule listings and scans).
