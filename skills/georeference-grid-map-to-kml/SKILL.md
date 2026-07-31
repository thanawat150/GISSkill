---
name: georeference-grid-map-to-kml
description: Georeference a map image or PDF that has no embedded spatial reference by using visible coordinate ticks, grid labels, graticules, or map-frame coordinates; trace the requested boundary or feature; validate the result; and export a KML in WGS 84. Use this skill when printed map coordinates are the only reliable spatial reference.
---

# Georeference Grid Map to KML

## Objective

Convert a non-georeferenced map image or PDF into a defensible spatial dataset by:

1. identifying the printed CRS, datum, zone, hemisphere, and coordinate axes;
2. deriving a pixel-to-map transformation from visible coordinate ticks or grid intersections;
3. creating a georeferenced raster without modifying the source;
4. tracing the requested boundary or feature;
5. validating geometry and positional consistency; and
6. exporting a KML in EPSG:4326 longitude, latitude order.

Do not stop after writing code. Run the workflow, inspect the outputs, perform QA/QC, and report the output paths and uncertainty.

## Use This Skill When

Use this skill when all or most of the following are true:

- the source is PNG, JPEG, TIFF, or PDF;
- the source has no usable GeoTIFF tags, world file, PDF geospatial metadata, or sidecar projection;
- the map shows coordinate ticks, grid lines, graticules, or labeled map-frame coordinates;
- a boundary, line, or point symbol must be extracted;
- the required delivery format is KML or KMZ.

## Do Not Use This Skill When

Do not use printed scale, a north arrow, place names, or visual similarity to a basemap as the sole georeferencing method.

Stop and request human input when:

- the datum is missing or ambiguous;
- the UTM zone or hemisphere cannot be determined;
- Easting and Northing values cannot be paired with reliable tick positions;
- the page has perspective distortion but no full two-dimensional control points;
- several candidate boundaries match the user's description;
- the requested feature is obscured, broken, or too low-resolution to trace defensibly.

Never silently assume that an old Thai map uses WGS 84. Older Thai sources may use Indian 1975 or another datum. Use the datum printed on the map or an authoritative accompanying source.

## Non-Negotiable Rules

- Never alter, crop over, rename, or overwrite the original input.
- Never invent coordinates, CRS parameters, zone, hemisphere, datum, or control points.
- Never use OCR as the first method. Read visible labels directly where possible; use OCR only as a confirmation or last-resort aid.
- Never use the scale bar as a location control. It may only be used as an independent scale check.
- Exclude legends, scale bars, north arrows, titles, and marginal graphics from feature extraction.
- Prefer command-line tools, Python APIs, or GDAL over GUI automation.
- Keep processing idempotent. If complete validated outputs already exist, do not overwrite them unless `overwrite` is explicitly true.
- KML coordinates must always be EPSG:4326 and ordered `longitude,latitude,altitude`.
- Record the method, control observations, residuals, warnings, and estimated uncertainty.
- Treat the extracted boundary as a map-derived interpretation, not a cadastral or legal boundary, unless the source itself has that authority.

## Required Job Contract

Create or read a UTF-8 `gis_job.json` before processing.

Required fields:

```json
{
  "job_id": "unique-job-id",
  "objective": "Georeference the source map and trace the requested feature",
  "input_paths": ["data/source/map.png"],
  "output_directory": "outputs/unique-job-id",
  "expected_outputs": [
    "map_georeferenced.tif",
    "feature_boundary.kml",
    "gcp_observations.csv",
    "qc_report.json",
    "preview.png"
  ]
}
```

Recommended fields:

```json
{
  "page": 1,
  "feature": {
    "name": "Target boundary",
    "geometry_type": "Polygon",
    "visual_description": "red outline with no fill"
  },
  "source_crs_hint": "WGS 84 / UTM zone 47N",
  "coordinate_observations": [],
  "render_dpi": 400,
  "simplify_pixels": 1.0,
  "overwrite": false,
  "human_review": true
}
```

`coordinate_observations` may initially be empty. Populate it after inspecting the source. Each observation must preserve its source and confidence:

```json
{
  "axis": "x",
  "pixel": 417.5,
  "value": 667500.0,
  "edge": "top",
  "label": "667500",
  "confidence": "high"
}
```

Use `axis: "y"` for Northing or latitude observations. For a full grid intersection, store both image pixel coordinates and both map coordinates.

## Expected Repository Layout

Use the existing repository structure when one exists. Otherwise create only the minimum required files:

```text
.
├── AGENTS.md
├── skills/
│   └── georeference-grid-map-to-kml/
│       └── SKILL.md
├── data/
│   ├── source/
│   └── working/
├── scripts/
│   └── georeference_grid_to_kml.py
├── outputs/
├── logs/
├── tests/
├── gis_job.json
└── README_TH.md
```

Do not scan the whole repository. Read only the relevant job file, this `SKILL.md`, and files required by the task.

## Tool Preference

Use the first available reliable option.

1. GDAL/OGR CLI:
   - `gdalinfo`
   - `gdal_translate`
   - `gdalwarp`
   - `ogr2ogr`
2. Python:
   - `numpy`
   - `opencv-python-headless`
   - `Pillow`
   - `PyMuPDF`
   - `pyproj`
   - `shapely`
   - `rasterio`
3. QGIS/PyQGIS only when already available and a QGIS-specific project is required.

Before execution, verify actual commands, package versions, and available APIs. Do not guess tool capability or parameters.

## Processing Workflow

### Phase 1 — Preflight and Source Inspection

1. Resolve every input path with `pathlib.Path`.
2. Confirm that the source exists and is readable.
3. Compute a checksum and record image dimensions, file type, page count, and color mode.
4. Inspect for existing spatial metadata:
   - GeoTIFF tags;
   - world files;
   - `.prj` files;
   - geospatial PDF metadata;
   - embedded CRS or GCPs.
5. If valid spatial metadata exists, do not derive a second transform from printed ticks unless the user explicitly requests verification.
6. Create a working copy or rendered page in `data/working/<job_id>/`.
7. Preserve the original source unchanged.

For PDF input, render only the requested page at 300–400 DPI by default. Use PyMuPDF or a verified PDF renderer. Record the render DPI because pixel-to-ground resolution depends on it.

### Phase 2 — Identify the Map Frame and CRS

1. Locate the rectangular map frame, not the page border.
2. Record the map-frame pixel bounds or corner coordinates.
3. Read the printed CRS, datum, zone, and hemisphere.
4. Normalize the CRS to an EPSG code where possible.
5. Validate axis meaning:
   - UTM Easting normally increases to the right;
   - UTM Northing normally increases upward;
   - image row values increase downward;
   - geographic KML output uses longitude first, latitude second.

Examples:

- `WGS 84` + `UTM Zone 47` + an unambiguous Thailand/Northern Hemisphere context → `EPSG:32647`.
- `Indian 1975 / UTM zone 47N` → use the matching Indian 1975 CRS, not EPSG:32647.
- `UTM Zone 47` without datum or hemisphere → ambiguous; stop.

### Phase 3 — Collect Coordinate Observations

Prefer direct visual reading over OCR.

For each labeled tick or grid intersection:

1. record the center pixel of the tick line or grid-line intersection;
2. record the printed coordinate value;
3. record which edge or grid line it came from;
4. record confidence and any ambiguity.

Target observations:

- minimum: two reliable X ticks and two reliable Y ticks after deskewing;
- preferred: at least three X ticks and three Y ticks;
- best: repeated ticks on opposite edges or full grid intersections distributed across the map.

Do not use the text glyph center as the control position. Use the actual tick or grid-line center.

If the map is rotated, estimate the map-frame angle and deskew before fitting the separable transform.

If the page has perspective distortion:

- rectify using reliable map-frame geometry only when the frame is demonstrably rectangular;
- then validate with repeated ticks;
- require full 2D control points if residuals indicate that a separable model is insufficient.

### Phase 4 — Fit the Pixel-to-Map Transformation

For an axis-aligned, non-perspective map with regular projected ticks, fit independent least-squares models:

```text
Easting  = ax * pixel_x + bx
Northing = ay * pixel_y + by
```

`ay` will normally be negative because image rows increase downward.

Use all reliable observations and calculate:

- coefficient values;
- residual for each observation;
- RMSE in pixels;
- RMSE in source CRS units;
- X and Y ground sampling distance;
- scale anisotropy.

Do not use a higher-order polynomial or thin-plate spline merely to reduce training residuals.

Use a first-order affine transform by default. Consider higher-order models only when:

- at least 10 well-distributed full 2D GCPs exist;
- physical scan distortion is evident;
- hold-out validation improves materially;
- the selected model is documented.

For ordinary printed map frames, a stable affine model is preferable to an overfit warp.

### Phase 5 — Georeference the Raster

Write a new GeoTIFF in the source map CRS.

When GDAL is available, use verified commands equivalent to:

```bash
gdal_translate \
  -of GTiff \
  -gcp <pixel_x> <pixel_y> <map_x> <map_y> \
  source.png working_with_gcps.tif
```

Then warp using the validated transformation and CRS:

```bash
gdalwarp \
  -order 1 \
  -s_srs EPSG:<source_epsg> \
  -t_srs EPSG:<source_epsg> \
  -r bilinear \
  -dstalpha \
  working_with_gcps.tif map_georeferenced.tif
```

The exact GCP command requires full two-dimensional GCPs. When only independent edge-axis ticks exist, compute the affine transform directly and write it with Rasterio/GDAL after deskewing. Do not fabricate the unknown coordinate component of a partial tick.

Use nearest-neighbor resampling for categorical maps. Use bilinear resampling for imagery.

### Phase 6 — Extract the Requested Feature

Restrict feature detection to the map-frame region.

For a colored outline such as a red boundary:

1. sample or estimate the target color in HSV or Lab color space;
2. create a conservative color mask;
3. remove antialiasing noise and isolated pixels;
4. label connected components;
5. reject components in legends and margins;
6. rank candidates by closedness, size, location, and visual description;
7. preserve the full candidate list in the QC report.

For an outlined polygon:

- derive the centerline of the outline using skeletonization or a midpoint between inner and outer contours;
- confirm that the skeleton is a single closed cycle;
- close only gaps shorter than a documented tolerance tied to line width;
- never bridge a large or ambiguous gap;
- transform the centerline vertices into the source projected CRS;
- simplify in projected units using approximately `0.5–1.5 ×` ground pixel size;
- preserve topology and important corners.

For a filled polygon, use its exterior contour and documented hole handling.

For a line feature, export a `LineString` unless the source clearly represents a closed area.

For points, use symbol centers and retain the symbol-detection uncertainty.

### Phase 7 — Geometry Validation

Validate before export:

- geometry type matches the request;
- polygon ring is closed;
- geometry is non-empty and valid;
- no self-intersections;
- no duplicate consecutive vertices;
- area and perimeter are plausible;
- geometry lies inside or overlaps the georeferenced map frame;
- coordinates fall within the expected regional bounds;
- source CRS to EPSG:4326 transformation succeeds;
- axis order is correct.

Use `shapely.make_valid` only when the cause is understood. Do not use it to hide a poor trace.

### Phase 8 — KML Export

Transform vector geometry from the source CRS to `EPSG:4326` using `pyproj.Transformer(..., always_xy=True)`.

KML requirements:

- UTF-8 XML;
- `longitude,latitude,altitude` coordinate order;
- altitude `0`;
- outer polygon ring oriented counter-clockwise where practical;
- meaningful feature name;
- red outline and transparent fill when tracing a red boundary;
- source and QC metadata in `ExtendedData`.

Recommended style:

```xml
<Style id="boundaryStyle">
  <LineStyle>
    <color>ff0000ff</color>
    <width>3</width>
  </LineStyle>
  <PolyStyle>
    <color>00000000</color>
    <fill>0</fill>
    <outline>1</outline>
  </PolyStyle>
</Style>
```

Remember that KML colors use `aabbggrr`, not RGB order.

### Phase 9 — QA/QC

Always produce a preview that overlays the extracted geometry on:

1. the original rendered map; and
2. the georeferenced raster or a coordinate grid.

Perform these checks:

- tick residual check;
- repeated opposite-edge tick check;
- map-frame closure check;
- X/Y pixel-size consistency;
- scale-bar consistency check;
- geometry-to-source overlay check;
- regional bounds check;
- KML re-open test.

Suggested status thresholds for a 1:50,000 printed map:

- `PASS`: tick residual ≤ 2 pixels and no CRS/geometry ambiguity;
- `WARNING`: residual > 2 and ≤ 5 pixels, or source line width causes material uncertainty;
- `FAIL`: residual > 5 pixels, CRS ambiguity, unresolved perspective, invalid geometry, or incorrect regional bounds.

Ground-distance thresholds must also be reported. Pixel thresholds alone are insufficient.

The source map scale and line thickness usually dominate final positional uncertainty. Do not report survey-grade accuracy from a 1:50,000 map.

### Phase 10 — Delivery

Create:

```text
outputs/<job_id>/
├── <stem>_georeferenced.tif
├── <feature_name>.kml
├── gcp_observations.csv
├── qc_report.json
├── preview_original.png
├── preview_georeferenced.png
└── processing.log
```

The QC report must include:

```json
{
  "job_id": "...",
  "source_file": "...",
  "source_sha256": "...",
  "source_crs": "...",
  "target_kml_crs": "EPSG:4326",
  "map_frame_pixels": {},
  "coordinate_observations": [],
  "transform": {},
  "rmse_pixels": {},
  "rmse_map_units": {},
  "ground_pixel_size": {},
  "feature_detection": {},
  "geometry": {},
  "warnings": [],
  "status": "PASS|WARNING|FAIL"
}
```

At completion, report:

- exact output paths;
- detected CRS and EPSG code;
- number of coordinate observations;
- RMSE and ground pixel size;
- feature type, area, and perimeter when applicable;
- warnings and estimated positional uncertainty;
- whether human review is still required.

## Reference Command Interface

When the repository does not already contain an implementation, create `scripts/georeference_grid_to_kml.py` with this interface:

```bash
python scripts/georeference_grid_to_kml.py --job gis_job.json --inspect
python scripts/georeference_grid_to_kml.py --job gis_job.json --run
python scripts/georeference_grid_to_kml.py --job gis_job.json --validate
```

Behavior:

- `--inspect`: inspect metadata, render a PDF page, create a map-frame/tick preview, and update the job draft without producing final geometry;
- `--run`: execute georeferencing, extraction, KML export, previews, and QC;
- `--validate`: re-open every output, validate raster CRS/transform, parse KML XML, check geometry, and return a non-zero exit code on failure.

Use structured logging and explicit error codes:

- `SOURCE_NOT_FOUND`
- `SOURCE_UNREADABLE`
- `CRS_AMBIGUOUS`
- `INSUFFICIENT_TICKS`
- `TICK_VALUE_AMBIGUOUS`
- `PERSPECTIVE_UNRESOLVED`
- `TRANSFORM_QC_FAILED`
- `MULTIPLE_FEATURE_CANDIDATES`
- `FEATURE_NOT_CLOSED`
- `GEOMETRY_INVALID`
- `KML_VALIDATION_FAILED`

Resume from completed phases where possible. Do not regenerate validated outputs unnecessarily.

## Worked Example — Bang Kachao Map

The following values describe one supplied example only. Never reuse them as defaults for another map.

Source characteristics:

- raster size: `1448 × 2048` pixels;
- map frame: approximately `x=103..1344`, `y=276..1516`;
- printed CRS: `WGS 84`, `UTM Zone 47`;
- interpreted CRS: `EPSG:32647` because the mapped area is unambiguously in Thailand/Northern Hemisphere.

Observed X ticks:

| Pixel X | Easting |
|---:|---:|
| 417.5 | 667500 |
| 848.5 | 670000 |
| 1279.5 | 672500 |

Observed Y ticks:

| Pixel Y | Northing |
|---:|---:|
| 533.5 | 1515000 |
| 964.0 | 1512500 |
| 1395.0 | 1510000 |

Derived approximate transform:

```text
Easting  = 5.80046404 * pixel_x + 665078.306
Northing = -5.80382988 * pixel_y + 1518095.86
```

Approximate ground pixel size:

- X: `5.8005 m/pixel`
- Y: `5.8038 m/pixel`

The red outline can be isolated inside the map frame as one dominant connected component. Use the outline centerline, not the outside edge, for the polygon trace.

The example result is suitable as a map-derived boundary with a documented positional warning. It is not a legal cadastral boundary.

## Completion Checklist

Before declaring success, confirm every item:

- [ ] original input preserved;
- [ ] CRS, datum, zone, and hemisphere verified;
- [ ] map frame identified;
- [ ] tick observations recorded;
- [ ] transform fitted and residuals reported;
- [ ] georeferenced raster created;
- [ ] target feature extracted from the map frame only;
- [ ] geometry validated;
- [ ] KML exported in longitude/latitude order;
- [ ] KML parsed and re-opened successfully;
- [ ] preview images created;
- [ ] QC report and log created;
- [ ] uncertainty and human-review status reported.
