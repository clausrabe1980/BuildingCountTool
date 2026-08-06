# 📚 Data Method Notes — Open Buildings (Pre-Downloaded Data)

## How Building Counts and Sizes Are Computed

This app uses **Google Open Buildings Temporal V1 and Open Buildings V3 datasets that
have been pre-downloaded and stored locally** for ward areas.

No Google Earth Engine (GEE) access is used at runtime. All processing is performed
locally on the prepared datasets.

Google's ML models originally detect buildings from satellite imagery and provide
either:

- per-pixel building presence probabilities (Temporal V1), or
- vector footprint polygons (V3).

The datasets included in this project are derived from those sources and prepared in
advance for ward-scale analysis.

---

## Metric CRS — how area is calculated

**All metric calculations use UTM 36S (EPSG:32736)**, set once in `app.R` as the
constant `METRIC_CRS`. This governs footprint area, ROI area, and the ROI simplification
tolerance.

### Why this matters

Geographic coordinates (EPSG:4326) cannot be used directly for area, so polygons are
reprojected to a metric CRS first. The choice of CRS is not neutral:

| CRS | Area accuracy at ~29.6°S | Verdict |
|---|---|---|
| Web Mercator (3857) | **+32.5%** — inflated by 1/cos²(lat) | Wrong |
| UTM 36S (32736) | −0.03% | Used |
| Geodesic on the ellipsoid (s2, no projection) | Reference | Marginally better, far slower |

Web Mercator preserves angles, not areas, and its area distortion grows with latitude.
UTM 36S was chosen over geodesic calculation because the accuracy difference is 0.03%
while the performance difference is substantial across ~1.6 million polygons.

Validated against the 2023 layer (211,511 polygons): geodesic 35,194,956 m²,
UTM 36S 35,183,244 m² (ratio 0.9997), Web Mercator 46,658,263 m² (ratio 1.3257).

### Correction history

Until 2026-08-06 the app used Web Mercator throughout. **All areas exported before that
date are overstated by approximately 32.5%** and should be divided by ~1.325. Building
counts and coverage percentage are unaffected — coverage is a ratio of two equally
distorted areas, so the distortion cancels.

Because the size filter operates on `area_m2`, filter thresholds also changed meaning.
Any calibration expressed as a size threshold must be re-derived.

---

## Temporal V1 Processing (Pre-Derived)

For Temporal V1, the original workflow (performed offline before packaging with the
app) was:

- Select the yearly building-presence mosaic
- Use the `building_presence` probability band
- Apply a probability threshold (default = 0.5)
- Convert the thresholded raster to polygons
- Treat each polygon as one detected building object

Polygon areas are **not** taken from the upstream export. They are recomputed by the
app at runtime from the geometry, in `METRIC_CRS`. Any area attribute present in the
source file is ignored.

These derived polygons are used for:

- building counts
- size histograms
- size filtering
- land coverage %

They are **derived shapes**, not original Google footprint vectors, and are stored
locally rather than generated at runtime.

---

## Storage format

Building layers are stored as **FlatGeobuf** (`.fgb`), converted from the original
GeoJSON exports on 2026-08-06.

The conversion is lossless for this pipeline: row counts, attribute columns, and
geometry types are preserved, and total area matches the GeoJSON source to the square
metre. FlatGeobuf stores full double-precision coordinates, where GeoJSON is text and
may be written with truncated decimals — so precision is equal or better.

Two consequences worth recording:

- **Row order changes.** FlatGeobuf writes features in packed Hilbert R-tree order —
  that ordering *is* the spatial index. Nothing in the pipeline depends on row order,
  but the `unique_id` sequence numbers in the Full XLSX are assigned at export time and
  will differ from any pre-conversion export. They were never stable join keys.
- **Reads are spatially filtered.** The app passes the ROI bounding box to GDAL as a
  `wkt_filter`, so only features near the ROI are read from disk. The exact intersection
  is still performed afterwards in `clip_to_roi()`, so results are unchanged — a
  bounding box always contains the polygon it bounds.

FlatGeobuf also preserves full-length attribute column names. This matters because
height is located by searching a list of candidate column names; shapefile format would
truncate names to 10 characters and cause height to be silently recorded as `NA`.

---

## Scale and Resolution Effects

Raster-to-polygon conversion depends on processing scale (metres per pixel). In the
original preprocessing workflow, a working scale equivalent to ~10 m resolution was
used.

Scale choice affects results:

**Smaller (finer) scale**

- more detailed polygon boundaries
- more small buildings/fragments detected
- higher counts possible

**Larger (coarser) scale**

- smoother polygons
- small buildings may merge or disappear
- lower counts

Building counts and sizes should therefore be interpreted as **scale-dependent
estimates**, not exact cadastral footprints.

Because preprocessing has already been completed, results are stable and reproducible
across users of this app.

---

## Temporal V1 vs V3 Datasets

### Temporal V1 — Open Buildings Temporal

- Multi-year dataset
- Originally raster probability layers
- Enables time-series analysis
- Required threshold + vectorisation (performed offline)
- Polygons are pre-derived and stored locally in this project

### V3 — Open Buildings V3 Polygons

- Single release dataset
- Precomputed building footprint polygons
- No time dimension
- Included locally as a comparison/reference layer

Counts differ between V1 and V3 because the datasets use different models, formats, and
post-processing pipelines.

### Registration and delineation

The two datasets are **not** spatially offset from one another. Measured over a dense
sample block: median centroid displacement 0.28 m east, 0.01 m north — effectively zero
against a ~3 m median nearest-neighbour distance.

Apparent misalignment when both layers are drawn together is a **delineation**
difference, not a georeferencing error. V3 splits attached structures that V1 merges
into a single footprint, so a V3 centroid sits within a portion of the corresponding V1
envelope rather than at its centre. This is the same phenomenon the dwelling-unit
correction exists to handle.

The V1:V3 count ratio is **not spatially constant** — approximately 2.24 in a dense
Hilton block against approximately 2.49 dataset-wide. A single global correction factor
is therefore an approximation whose error depends on the local mix of attached and
freestanding housing.

Note also that the Esri satellite basemap may itself sit several metres from ground
truth. This affects visual interpretation only; no calculation in the app references
the basemap.

---

## Data Access and Reproducibility

- All Open Buildings data used by the app is **pre-downloaded and stored locally**
- No Google Earth Engine calls are made by the app
- No API keys or cloud credentials are required
- No authentication steps are needed
- All users run the same prepared datasets
- Results are reproducible across machines given the same repository contents and the
  same `Data/` folder

Heavy cloud computation was performed only during the original offline data preparation
stage, not during app execution.
