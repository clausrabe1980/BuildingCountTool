# Building Count Tool

An interactive R Shiny application for exploring Google Open Buildings data within
user-defined regions of interest (ROI). Draw an area or select a predefined boundary,
then analyse building counts, footprint sizes, heights, and land coverage from 2016
to 2023.

Developed for the Hilton Local Area Plan: Growth & Fiscal Impact Model. See
`Method notes/building_count_methods_note.Rmd` for the full technical description of
data sources, processing pipeline, and known limitations.

---

## ⚠️ Areas changed on 2026-08-06 — read this before using any export

Footprint areas were previously computed in Web Mercator (EPSG:3857), which inflates
area by `1 / cos²(latitude)` — approximately **32.5% at this latitude**. The app now
uses UTM 36S (EPSG:32736), accurate to ~0.03%.

**Any export produced before 2026-08-06 overstates areas by ~32.5%.** Affected fields:

| Field | Affected? |
|---|---|
| `area_m2` (Full XLSX) | Yes — divide by ~1.325 |
| `total_built_area_m2` (Summary XLSX) | Yes — divide by ~1.325 |
| `roi_area_m2` | Yes — divide by ~1.325 |
| `building_count` | No |
| `cover_pct` | No — both numerator and denominator were equally inflated |
| `height_m` | No |

The exact factor varies slightly with ROI latitude (1.324–1.326 across the study area).
Any gross floor area estimate derived from footprint area is affected in the same
proportion.

**Size-filter thresholds also change meaning.** A 50 m² filter under the old code
excluded buildings below ~38 m² of true footprint. Any calibration expressed as a size
threshold — including V1/V3 alignment — must be re-derived against corrected areas. The
resulting *count ratio* should be unchanged, since both datasets scaled equally.

---

## ⚠️ The data is NOT in this repository

The datasets total roughly **900 MB** and are excluded via `.gitignore`. **Cloning this
repo alone will not give you a working app.** You must obtain the `Data/` folder
separately and place it in the project root.

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
│   │   ├── V1_2016_UMN_Functional_Areas_3_withHeight.fgb
│   │   ├── V1_2017_UMN_Functional_Areas_3_withHeight.fgb
│   │   ├── ...  (one per year through 2023)
│   │   └── V1_2023_UMN_Functional_Areas_3_withHeight.fgb
│   ├── V3 Shapefile_large/
│   │   └── V3_2023_UMN_Functional_Areas_3.fgb
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
length(list.files("Data/V1 Temporal_large", pattern = "\\.fgb$"))   # expect 8
```

---

## Data format: FlatGeobuf

Building layers are stored as **FlatGeobuf** (`.fgb`), converted from the original
GeoJSON exports on 2026-08-06. Two reasons:

- **Size** — ~1.45 GB of GeoJSON became ~900 MB.
- **Speed** — `.fgb` carries a packed Hilbert R-tree spatial index, so the app reads
  only features intersecting the ROI bounding box instead of parsing every polygon.
  ROI selection went from minutes to near-instant.

The conversion was verified as lossless: identical row counts, identical attribute
columns, geometry type preserved, and total area matching to the square metre.
FlatGeobuf reorders features on write (that is the index), so **row order differs from
the source GeoJSON**. Nothing in the app depends on row order.

`sf::st_read()` detects the format from the file, so no reader code is format-specific.
Reverting to GeoJSON would require only changing the extensions in `TEMPORAL_FILES` and
`V3_FILE`.

### Regenerating the .fgb files

If the GeoJSON exports are ever refreshed from Earth Engine, reconvert with:

```r
library(sf)
srcs <- c(
  list.files("Data/V1 Temporal_large",  pattern = "\\.geojson$", full.names = TRUE),
  list.files("Data/V3 Shapefile_large", pattern = "\\.geojson$", full.names = TRUE)
)
for (s in srcs) {
  d <- sub("\\.geojson$", ".fgb", s)
  if (file.exists(d)) next
  st_write(st_read(s, quiet = TRUE), d, quiet = TRUE)
}
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
  - *Summary XLSX* — annual aggregates (count, built area, % cover) plus ROI metadata.
    Respects the current slider settings.
  - *Full XLSX* — one row per building polygon (`unique_id`, `year`, `area_m2`,
    `height_m`) across two sheets, `Buildings` and `V3_control`. **Unfiltered** —
    slider settings do not apply.

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
- `sf` must be built against GDAL 3.1 or newer for FlatGeobuf support. Any recent
  CRAN binary satisfies this. Check with `sf::sf_extSoftVersion()`.

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
Either `Data/` is missing entirely, or the filenames have drifted. Compare
`list.files("Data/V1 Temporal_large")` against the paths near the top of `app.R`. Note
the app now expects `.fgb`, not `.geojson`.

**"Unable to open datasource" or an unknown-driver error**
GDAL is too old for FlatGeobuf. Check `sf::sf_extSoftVersion()["GDAL"]` — 3.1 or newer
is required. Reinstall `sf` from CRAN binaries.

**UMN areas render but some polygons are missing**
`app.R` points at one specific version of the regions shapefile. The `Data/Regions/`
folder contains several (`_1` through `_7f`). Confirm the line near the top of `app.R`
points at the version you intend.

**"cannot remove prior installation of package X"**
A package was being reinstalled while loaded. Restart R (Ctrl+Shift+F10), then
reinstall with nothing attached. If it persists, delete any `00LOCK*` folders in your
library directory.

**Two ROIs return the same numbers**
Should not occur, but if it does the layer cache key is at fault. The cache is keyed by
year *and* ROI bounding box (`roi_key()` in `app.R`); a key collapsing to year alone
would serve one ROI's data for another. Restart the app to clear the cache and report it.

---

## Notes and known limitations

- `unique_id` in the Full XLSX export is generated sequentially at export time as
  `{year}_{sequence}`. **These IDs are not stable across export sessions** — the same
  ROI exported twice will produce different sequence numbers. Do not use them as join
  keys against earlier exports.
- Building height is found by searching a priority list of candidate column names.
  If none match, height is silently recorded as `NA`. No log records which column was
  chosen. (FlatGeobuf preserves full column names, so this list still matches. Shapefile
  would *not* — it truncates names to 10 characters and would silently return all `NA`.)
- The presence confidence threshold (0.50) is baked into the exported data and cannot
  be changed within the app.
- Merged polygons: where the detection algorithm joins attached structures, one polygon
  may represent multiple dwellings. Handled downstream in the analysis workbook, not here.
- **The V1:V3 count ratio is not spatially constant.** Sampled at ~2.24 in a dense
  Hilton block against ~2.49 dataset-wide. A single global dwelling-correction factor
  is therefore an approximation, and will over- or under-correct depending on the mix
  of attached and freestanding housing in the ROI.
- V1 and V3 polygons are **not misregistered** — median centroid offset measured at
  0.28 m east, 0.01 m north. Apparent offset on the map is different delineation of the
  same buildings (V3 splits what V1 merges), not a coordinate error. The Esri basemap
  itself may sit several metres off ground truth; nothing in the tool references it.

---

## Repository layout

| Path | Contents |
|---|---|
| `app.R` | **The application.** Sole current version. |
| `Method notes/` | Technical methods note (Rmd + docx) |
| `DataMethodNotes.md` | How counts and sizes are derived |
| `Data analysis/` | Exported workbooks and downstream analysis |
| `HiltonBuildingsTool.Rmd` | **Stale.** Literate walkthrough of a superseded app version, with hardcoded macOS paths. Archive only — do not run. |
| `Data/` | Not in repo — see above |

Earlier app versions (`SS/`, `Claus version/`) were removed on 2026-08-06 and are
recoverable from Git history: `git checkout 1e6696b -- SS "Claus version"`.
