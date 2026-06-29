# CineBOTrends — Dashboard

**Real-Time Box Office Intelligence** for Indian cinema. A static, framework-free
web dashboard that visualises live box-office data: collections, occupancy, ticket
sales, and theatre-, city- and state-level breakdowns, plus historical trends.

This is the **public** dashboard repo. It reads pre-built JSON from `data/`.
The JSON is produced from your **private data-collector** repo (see *Data flow* below).

Brand: amber `#FFB000` · dark theme · **Retroica** display font · **flaticon UIcons**.

---

## Quick start

```bash
# 1. Build the dashboard data from your collector output
python3 build_data.py /path/to/datacollector

# 2. Serve the folder (browsers block fetch() on file://, so a server is required)
python3 -m http.server 8000

# 3. Open
#    http://localhost:8000
```

That's it — no build step, no npm, no framework.

---

## How it fits together

```
┌─────────────────────────┐        build_data.py        ┌──────────────────────────┐
│  data-collector (PRIVATE)│  ───────────────────────▶   │  cinebotrends (PUBLIC)   │
│  scrapes & writes        │   reads finaldetailed.json  │  static dashboard        │
│  advance/data/<date>/... │   aggregates → data/        │  reads data/*.json       │
└─────────────────────────┘                              └──────────────────────────┘
```

* The collector stays private (it contains scraping logic and source config).
* `build_data.py` is the bridge: it reads the collector's output and writes the
  compact JSON this dashboard serves. It never modifies the collector.
* Commit the generated `data/` folder to this public repo so it deploys with the site.

### Data flow / sync

The collector writes, per run:

```
<collector>/advance/data/<YYYYMMDD>/finaldetailed.json   # show-level rows (the source of truth)
<collector>/daily/data/<YYYYMMDD>/finaldetailed.json     # appears once daily mode runs
```

`build_data.py` turns those into:

```
data/manifest.json                      # modes, tracked dates, run schedule
data/<mode>/<date>/national.json        # home grid + hero KPIs (all movies, ~36 KB)
data/<mode>/<date>/m/<slug>.json        # full drill-down for one movie (lazy-loaded)
data/<mode>/history/<slug>.json         # day / city / state / format-wise history
```

Re-run `build_data.py` whenever the collector produces new data, then commit `data/`.
You can automate this in the collector's CI (e.g. a GitHub Action that runs the
builder and pushes `data/` to this repo).

---

## The Retroica font

The brand display face is **Retroica** (free for personal use):
<https://www.dafont.com/retroica.font>

Because of licensing it isn't bundled. To enable it:

1. Download Retroica from dafont.
2. Place the file in `fonts/` as `Retroica.ttf` (and/or `Retroica.woff2`).

Until then the dashboard falls back to **Space Grotesk** for the wordmark and big
numbers, so everything still renders cleanly. Body text uses **Hanken Grotesk** and
data tables use **JetBrains Mono**, both loaded from Google Fonts at runtime.

---

## Screens

| Screen | What it shows |
|---|---|
| **Splash** | Logo + "Real-Time Box Office Intelligence", fade-and-scale, auto-advances to Home. |
| **Home** | Fixed header · "Box Office Tracker" hero with a live ticker · All-Movies grid (6 / 4 / 1 per row) · social KPIs · footer with newsletter. |
| **Movie Details** | Movie info + three tabs: **Advance**, **Daily** (default), **Historical**. |
| **Advance / Daily** | Tracked-date chips → 8 KPI summary cards → Top 20 Cities → State cards → State details → City details → Theatre accordion with show-timings → Format Summary cards. |
| **Historical** | Day-wise, City-wise, State-wise and Format-wise performance tables. |

Drill-downs are routed (`#/movie/<slug>/advance/<date>/state/<state>/city/<city>`),
so the browser back button and shareable links work.

---

## Notes on the data

* **Grouping.** The collector lists each format/language variant separately
  (e.g. `Peddi (2D - Telugu)`). The dashboard groups variants under one title so the
  **Format Summary** (2D / 3D / IMAX / 4DX / ICE / Dolby Cinema / Others) is meaningful.
* **Occupancy** is recomputed as `sold / capacity` (the stored string is unreliable).
* **Houseful** = occupancy ≥ 98%, **Fast filling** = 50–98%, matching the collector.
* **Daily & Historical** light up automatically as the collector produces more dates.
  With a single date, Daily shows an empty state and Historical shows one day.
* **Movie metadata** (cast, director, runtime, certification) isn't present in the
  collector output, so those fields show "Not in dataset". If you later add a metadata
  source, extend `build_data.py`.

---

## Posters (BookMyShow images)

Cards and the movie-details page show real posters when you provide them; otherwise
they fall back to a styled initials placeholder (and any image that fails to load
falls back automatically too).

BookMyShow serves each poster from a predictable path, where the two sizes differ
only by one segment and share the same filename "id":

```
thumbnail (portrait, used on cards + detail poster):
  https://assets-in.bmscdn.com/iedb/movies/images/mobile/thumbnail/xlarge/<id>.jpg
listing  (wide, used as the blurred detail backdrop):
  https://assets-in.bmscdn.com/iedb/movies/images/mobile/listing/xxlarge/<id>.jpg

<id> = "<title-slug>-<event-code>-<timestamp>"
       e.g.  balaramana-dinagalu-et00478884-1782106150
```

To add posters, create **`posters.json`** in the dashboard root mapping each movie
title (or slug) to its BMS id:

```json
{
  "Maa Inti Bangaaram": "balaramana-dinagalu-et00478884-1782106150",
  "Peddi": "peddi-et00xxxxxx-17xxxxxxxx"
}
```

`build_data.py` expands each id into the thumbnail and listing URLs automatically.
You can also pass `{"thumb": "...", "bg": "..."}` for fully custom URLs.

Every run writes **`posters.template.json`** — a ready-to-fill list of every tracked
title — so you only paste in the ids. Rename it to `posters.json` once filled.
(The shipped `posters.json` seeds one title as a live example.)

> Best practice: have the **collector** record each movie's BMS id while scraping, then
> emit a `posters.json` next to its data — the dashboard picks it up with zero manual work.

### Posters + metadata from getDiscoveryData (recommended)

BookMyShow's `getDiscoveryData` endpoint returns posters **and** metadata per region in
one call — `defaultPosterImage`, `defaultEventCode`, `genres`, and `variants` with
`runTime`, `censorRating`, `language`, `format`. This fills the poster plus the Genre /
Runtime / Certification fields on the detail page.

While you track the 9 shards, also hit it once per region and save each response:

```
GET https://cinemas.bookmyshow.com/api/getDiscoveryData?region=BANG  ->  discovery/BANG.json
```

Then convert all region files into `posters.json` + `metadata.json`:

```bash
python3 tools/discovery_to_meta.py discovery/      # folder of <REGION>.json
# or a single file:
python3 tools/discovery_to_meta.py getDiscoveryData.json
```

`build_data.py` reads both automatically and attaches genre/runtime/certification (and the
real poster + blurred backdrop) to each movie. Titles are matched case-insensitively, so
more regions = more movies covered. The shipped `posters.json` / `metadata.json` were
generated from one Bangalore response as a working example.



`tools/posters_from_bms.py` (run inside your collector) resolves posters from the
`movies-data/showtimes-by-event` endpoint, using only `ImagePortraitUrl`, `bannerUrl`
and `imageUrl` when present. Check the response first, then build:

```bash
python3 tools/posters_from_bms.py --inspect ET00478884      # dump one event's JSON + show image fields
python3 tools/posters_from_bms.py --codes eventcodes.json   # write posters.json (skips movies with no image)
```

`eventcodes.json` is a `title → ET-code` map — capture `EventCode` in your by-venue
parser (it's on each `Event` alongside `EventTitle`). The endpoint is Cloudflare-gated,
so it only runs from the collector's session; copy the exact region/date params from your
working request into `BASE_PARAMS`.

---

## Deploy (GitHub Pages)

This is fully static. Push the repo (including `data/` and `fonts/`) and enable
GitHub Pages on the default branch. No configuration needed.

---

## Project structure

```
cinebotrends/
├── index.html          # app shell + splash
├── css/app.css         # design system (dark marquee theme)
├── js/app.js           # helpers, data layer, router, shared chrome
├── js/screens.js       # all screens
├── build_data.py       # collector output → data/
├── posters.json        # title → BookMyShow image id (optional)
├── data/               # generated JSON (commit this)
└── fonts/              # drop Retroica here
```
