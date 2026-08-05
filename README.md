# Building Count Tool

An interactive R Shiny application for exploring Google Open Buildings data within
user-defined regions of interest (ROI). Draw an area or select a predefined boundary,
then analyse building counts, footprint sizes, heights, and land coverage from 2016
to 2023.

Developed for the Hilton Local Area Plan: Growth & Fiscal Impact Model. See
`Method notes/building_count_methods_note.Rmd` for the full technical description of
data sources, processing pipeline, and known limitations.

---

## ⚠️ The data is NOT in this repository

The satellite-derived datasets total roughly **1.7 GB** and are excluded via
`.gitignore`. **Cloning this repo alone will not give you a working app.** You must
obtain the `Data/` folder separately and place it in the project root.

**Where the data lives:** _[FILL THIS IN — e.g. OneDrive link, shared drive path,
or "ask Claus"]_

### Required folder structure

```
BuildingCountTool/
├── app.R                    <- the application (this repo)
├── README.md
├── Method notes/
├── Data/                    <- NOT in repo; add manually
│   ├── V1 Temporal_large/
│   │   ├── V1_2016_UMN_Functional_Areas_3_withHeight.geojson
│   │   ├── V1_2017_UMN_Functional_Areas_3_withHeight.geojson
│   │   ├── ...  (one per year through 2023)
│   │   └── V1_2023_UMN_Functional_Areas_3_withHeight.geojson
│   ├── V3 Shapefile_large/
│   │   └── V3_2023_UMN_Functional_Areas_3.geojson
│   ├── Wards/
│   │   └── Municipal_Wards_2021.shp   (+ .shx .dbf .prj .cpg)
│   └── Regions/
│       └── UMN_Functional_Areas_7f.shp   (+ .shx .dbf .prj .cpg)
```

Shapefiles need their companion files (`.shx`, `.dbf`, `.prj`) in the same folder —
a lone `.shp` will not open.

### Verify before running

```r
file.exists("Data/Wards/Municipal_Wards_2021.shp")
file.exists("Data/Regions/UMN_Functional_Areas_7f.shp")
length(list.files("Data/V1 Temporal_large", pattern = "\\.geojson$"))   # expect 8
```

---

## Features

- **ROI selection** — draw a custom polygon, click a municipal ward, or click a UMN
  functional area
- **Dual filtering** — live range sliders on both footprint area and building height;
  filters apply to the map, the time-series, and the exports simultaneously
- **Time-series** — annual building counts and total built area, 2016–2023, with the
  V3 2023 count overlaid as a calibration reference
- **Histograms** — footprint area and building height distributions
- **Two export formats:**
  - *Summary XLSX* — annual aggregates (count, built area, % cover) plus ROI metadata
  - *Full XLSX* — one row per building polygon (`unique_id`, `year`, `area_m2`,
    `height_m`) across two sheets, `Buildings` and `V3_control`

The Full XLSX export is what feeds the Building Analysis workbook.

---

## Data sources

| Dataset | Role |
|---|---|
| Google Open Buildings **Temporal V1** | Primary source. Annual polygons 2016–2023, each with footprint area and height. Exported at a presence confidence threshold of 0.50, fixed upstream. |
| Google Open Buildings **V3** | 2023-only snapshot using a more granular classification. Detects individual structures rather than merged footprints, so typically yields higher counts. Used as a calibration reference. |
| **Municipal Wards** (2021) | Boundary layer for ROI selection |
| **UMN Functional Areas** | Planning sub-areas; primary unit of analysis |

No Google Earth Engine, `rgee`, `reticulate`, or Python setup is required — all
Earth Engine access was removed in favour of pre-exported local files.

---

## Setup

### Prerequisites

- R 4.2 or newer
- RStudio (recommended)
- On Windows: **Rtools** may be needed if `sf` has to build from source
  (https://cran.r-project.org/bin/windows/Rtools/)

### Install packages

```r
install.packages(c(
  "shiny", "leaflet", "leaflet.extras", "sf",
  "jsonlite", "geojsonsf", "htmlwidgets", "plotly", "openxlsx"
))
```

If `sf` fails on macOS, run `xcode-select --install` in Terminal, then reinstall.

### Clone

```bash
git clone https://github.com/clausrabe1980/BuildingCountTool.git
cd BuildingCountTool
```

Then add the `Data/` folder as described above.

### Run

Open `app.R` in RStudio and click **Run App**, or:

```r
shiny::runApp("app.R")
```

You should see `Listening on http://127.0.0.1:XXXX`.

---

## Troubleshooting

**App hangs on launch, spinner never resolves**
Check for uncommented `install.packages()` or `remotes::install_github()` calls at the
top of `app.R`. `runApp()` sources the whole file, and `install_github()` opens an
interactive prompt that cannot be answered while the app is starting. These lines are
commented out in the current version — keep them that way. Install packages from the
Console instead.

**Layers load but the map is empty**
Almost always a path problem. `runApp()` sets the working directory to wherever the app
file sits, so `app.R` must be in the project root alongside `Data/`. Check with
`getwd()` and the `file.exists()` calls above.

**"Cannot open ... The file doesn't seem to exist"**
Either `Data/` is missing entirely, or the filenames have drifted. The GeoJSON exports
have been renamed at least once (timestamped → `_withHeight`), so compare
`list.files("Data/V1 Temporal_large")` against the paths near the top of `app.R`.

**UMN areas render but some polygons are missing**
`app.R` points at one specific version of the regions shapefile. The `Data/Regions/`
folder contains several (`_1` through `_7f`). Confirm the line near the top of `app.R`
points at the version you intend.

**"cannot remove prior installation of package X"**
A package was being reinstalled while loaded. Restart R (Ctrl+Shift+F10), then
reinstall with nothing attached. If it persists, delete any `00LOCK*` folders in your
library directory.

**ROI selection is slow (minutes)**
The cost is in the geometry clip, not file loading. A bounding-box pre-filter before
the exact intersection would cut this substantially. Not yet implemented.

---

## Notes and known limitations

- `unique_id` in the Full XLSX export is generated sequentially at export time as
  `{year}_{sequence}`. **These IDs are not stable across export sessions** — the same
  ROI exported twice will produce different sequence numbers. Do not use them as join
  keys against earlier exports.
- Building height is found by searching a priority list of candidate column names.
  If none match, height is silently recorded as `NA`. No log records which column was
  chosen.
- The presence confidence threshold (0.50) is baked into the exported data and cannot
  be changed within the app.
- Merged polygons: where the detection algorithm joins attached structures, one polygon
  may represent multiple dwellings. Handled downstream in the analysis workbook, not here.

---

## Repository layout

| Path | Contents |
|---|---|
| `app.R` | **Current version.** Height support, dual export. |
| `Claus version/` | Older working copies (`app2.R`, `app3.R`) — superseded |
| `SS/` | Earlier iterations (`app.R` through `app7.R`) — archive |
| `Method notes/` | Technical methods note (Rmd + docx) |
| `Data/` | Not in repo — see above |

> **Housekeeping note:** there are currently twelve app versions across three folders.
> Worth consolidating to a single `app.R` with the rest either deleted or moved to an
> `archive/` folder, now that Git provides the version history.
