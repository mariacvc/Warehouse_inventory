# Southern California Warehouse Inventory
### Stage 1 — Geospatial and Operational Inventory of Warehouses

Geospatial and operational inventory of large warehouse and distribution
facilities in Southern California, and the spatial characterization of their
distribution over space and time. This repository implements **Task 2 —
Characterization of freight facility locations** of the project *Empirical
Assessment of Land Use and Other Policy Impacts on Warehousing Location Choices
in Southern California* (Caltrans Agreement 65A1345; National Center for
Sustainable Transportation).

- **PI:** Miguel Jaller, Civil & Environmental Engineering, UC Davis
- **Lead Researcher:** Maria C. Valencia-Cardenas, Graduate Student Researcher
- **Institution:** Institute of Transportation Studies, UC Davis (ITS-Davis)
- **Status:** Research code — draft deliverable
---

## What this notebook does

`Warehouse_Inventory_Caltrans.ipynb` builds a reproducible facility inventory by
fusing open building-footprint sources, classifying buildings, enriching them
with land-use, property, and employment data, validating against the Census
County Business Patterns, and characterizing the spatial pattern of the result.

1. **Study area** — TIGER/2018 county boundaries for the six study counties.
2. **Footprint sources** — three vector footprint sources (Microsoft US Building
   Footprints, Overture Maps, OpenStreetMap) plus USDA NAIP imagery for roof
   detection via Google Earth Engine.
3. **Fusion & classification** — deduplication (IoU >= 0.3), multi-signal
   semantic classification (`confirmed` / `candidate` / `excluded`), an
   OSM-derived land-use spatial filter, and a functional subtype typology.
4. **Enrichment** — SCAG land-use layers (public), CoStar property records
   (construction year; **proprietary/licensed — see Data Sources & Licensing**),
   and LEHD LODES employment (NAICS 48-49, 2003-2023; public).
5. **Validation** — ZIP-level comparison against Census CBP (NAICS 492 + 493).
6. **Spatial characterization** — Nearest Neighbor Index, quadrat counts,
   Ripley's K/L, weighted KDE, centrographic analysis (weighted mean center +
   standard deviational ellipse), and spatial/temporal anomaly detection.

## Study area

Six counties: Los Angeles, Orange, Riverside, San Bernardino, Ventura, and
Imperial. Footprint threshold: **>= 50,000 ft²** (4,645.15 m²). Temporal frame:
**2003-2023**.

## Two population counts (read this before citing numbers)

The notebook uses two related but distinct counts. They are not
interchangeable:

| Count | Meaning | Used for |
|-------|---------|----------|
| **Final inventory** | all included facilities (confirmed + high-confidence candidates, after manual review) | headline facility count, maps, all spatial analyses |
| **Temporal subset** | facilities with a reliable construction / operation-start year (after removing the LODES8 vintage artifact and LODES7 truncation) | year-by-year trajectory, anomaly detection |

> **Important:** all *spatial* analyses (NNI, quadrat, Ripley's K, centrographic)
> are run on the **final inventory** population so the spatial results and the
> headline count describe the same set. See `spatial_pattern_additions.py`,
> CELL C. The temporal subset is necessarily smaller because only datable
> buildings can be placed in time — this is a coverage limitation, not an
> inconsistency.

---

## Repository Structure

```
warehouse-inventory/
│
├── Warehouse_Inventory_Caltrans.ipynb   # Main analysis notebook
│
├── data/                        # Downloaded raw data (NOT tracked in git)
│   ├── California.geojson.zip   # Microsoft building footprints (cached)
│   ├── la_buildings.geojson     # Overture buildings — Los Angeles
│   ├── or_buildings.geojson     # Overture buildings — Orange
│   ├── ri_buildings.geojson     # Overture buildings — Riverside
│   ├── sb_buildings.geojson     # Overture buildings — San Bernardino
│   ├── ve_buildings.geojson     # Overture buildings — Ventura
│   └── imp_buildings.geojson     # Overture buildings — Imperial
│
├── Shapefiles/                  # OSM-derived land-use layers (NOT tracked in git)
│   ├── LA_osm_lulc_training.shp
│   ├── OR_osm_lulc_training.shp
│   ├── RI_osm_lulc_training.shp
│   ├── SB_osm_lulc_training.shp
│   ├── VE_osm_lulc_training.shp
│   └── IMP_osm_lulc_training.shp
│
├── outputs/                                   # Inventory outputs
│   ├── warehouse_final_inventory.gpkg         # final inventory (attributes + geometry)
│   ├── warehouse_final_with_years.gpkg        # + LODES employment panel & onset fields
│   ├── warehouse_inventory_clean.csv          # cleaned, shareable attribute table
│   └── data_dictionary.md                     # column definitions for the CSV
│
├── environment.yml              # Conda environment specification
└── README.md                    # This file
```

> **Note:** Raw data files (`.geojson`, `.zip`, `.shp`) and **all CoStar data**
> are excluded from version control. See **Data Sources & Licensing** for access
> and redistribution terms.

---

## Methodology

The pipeline has four main stages:

```
Step 1: Define Study Area
        └─ TIGER/2018 county boundaries (six counties) via Google Earth Engine

Step 2: Collect Building Footprints
        ├─ USDA NAIP imagery (roof detection via GEE — imagery, not polygons)
        ├─ Overture Maps (rich semantic schema)
        ├─ Microsoft US Building Footprints (geometric accuracy)
        └─ OpenStreetMap (named facilities, semantic tags)

Step 3: Classify Buildings
        ├─ Semantic tags (OSM building=warehouse/industrial)
        ├─ Overture schema (subtype, class)
        ├─ Name/brand matching (Amazon, FedEx, Prologis, etc.)
        ├─ OSM point tag carriers (buffered spatial join)
        └─ Geometric shape metrics (rectangularity, elongation)

Step 4: Land Use Filter
        └─ Spatial join to OSM-derived LULC county shapefiles
```

### Classification Label System

Every building in the output carries one of three labels:

| Label | Meaning | `needs_review` |
|-------|---------|----------------|
| `confirmed` | Strong positive signal — high-confidence warehouse/industrial | `False` |
| `candidate` | Passes all filters but lacks a positive signal — review shape metrics | `True` |
| `excluded` | Matched an exclusion rule — retained for audit trail | `False` |

The `confidence_note` column records the exact rule that triggered each label
(e.g., `overture_class:warehouse`, `name_keyword:amazon`, `lu_exclude:Residential`).

---

## Data Sources & Licensing

| Source | Description | Access / License |
|--------|-------------|------------------|
| [USDA NAIP](https://developers.google.com/earth-engine/datasets/catalog/USDA_NAIP_DOQQ) | ~0.6 m aerial imagery (RGBN); roof detection | Public — Google Earth Engine |
| [Overture Maps Buildings](https://docs.overturemaps.org/guides/buildings/) | Consolidated footprints + semantics | Open (CDLA-Permissive / ODbL) — `overturemaps` CLI |
| [Microsoft Building Footprints](https://github.com/microsoft/USBuildingFootprints) | Computer-vision roof polygons | Open (ODbL) — Azure Blob (~5 GB) |
| [OpenStreetMap](https://wiki.openstreetmap.org/wiki/Key:building) | Named facilities, building tags | Open (ODbL) — `osmnx` |
| [TIGER/2018 Counties](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) | County boundaries | Public domain — Google Earth Engine |
| [LEHD LODES](https://lehd.ces.census.gov/data/) | Workplace employment, NAICS 48-49 (CNS08), 2003-2023 | Public domain — Census LEHD |
| SCAG land use | Parcel-level land-use layers | Public — SCAG |
| **CoStar** | Commercial property records (construction year, RBA, property type) | **PROPRIETARY — paid subscription; NOT public, NOT redistributed** |

The open footprint, boundary, employment, and validation sources above are
publicly accessible, reproducible, and scientifically defensible for peer
review and statewide scaling.

**CoStar is the exception and requires explicit handling:**

- CoStar is a **commercial, licensed dataset**. It is **not** open or public,
  and **no raw CoStar data is included in this repository**. Reproducing the
  enrichment step requires the user's own CoStar subscription.

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/mariacvc/warehouse-inventory.git
cd warehouse-inventory
```

### 2. Set Up the Environment

```bash
conda env create -f environment.yml
conda activate geo
```

### 3. Configure Google Earth Engine

```python
import ee
ee.Authenticate()   # Run once per machine
ee.Initialize()
```

### 4. Download Overture Building Data

Run once per county. Large counties (LA, SB, RI) may take 15–30 minutes.

```bash
# Example for Los Angeles
overturemaps download \
  --bbox=-118.9448,33.7036,-117.6463,34.8233 \
  -f geojson --type=building \
  -o data/la_buildings.geojson
```

Or use the tiled downloader in the notebook for more reliability:

```python
tiles = download_overture_tiled("data/la", bbox=bboxes["la"], nx=4, ny=4)
la_overture_raw = merge_overture_tiles(tiles)
```

### 5. Provide CoStar Data (optional, licensed)

The enrichment step expects CoStar exports under `data/`. These are **not**
provided; supply your own under a valid CoStar license. The notebook runs
without them, but `year_built` and CoStar-derived attributes will be empty.

### 6. Set Your Path

In the notebook Setup cell, update:

```python
path = '/your/local/project/path/'
```

### 7. Run the Notebook

Execute cells sequentially. The Results section at the bottom runs the full pipeline.

---

## Output

The primary outputs are two GeoPackages and a cleaned CSV (see Repository
Structure). Column definitions for the shareable CSV are in
[`data_dictionary.md`](outputs/data_dictionary.md).

**Selected columns:**

| Column | Type | Description |
|--------|------|-------------|
| `facility_id` | str | Stable surrogate primary key (complete) |
| `geometry` | Polygon | Building footprint (EPSG:4326) |
| `source` | str | Data source: `osm`, `overture`, or `ms` |
| `label` | str | `confirmed`, `candidate`, or `excluded` |
| `subclass` | str | Functional subtype (FI-IND, U-DF, R-DC, R-RL, ambiguous) |
| `confidence_score` / `confidence_tier` | num / str | Rule-based confidence (0–100; binned) |
| `area_ft2` | float | Building footprint area in ft² |
| `scag_land_use_class` | str | SCAG parcel land-use class |
| `ops_start_year` | int | Resolved start year (canonical; see `year_source`) |
| `year_built` | int | **CoStar** construction year (licensed; ~34% coverage) |
| `lodes_onset_year` | int | First LODES logistics-employment year |
| `temporal_use` | str | Onset validity (`usable` / `lodes8_artifact` / `lodes7_truncation` / `no_onset`) |

### QGIS Quick-Start Filters

```
Confirmed warehouses:              "label" = 'confirmed'
Candidates needing review:         "needs_review" = 1
Excluded buildings:                "label" = 'excluded'
Names flagged for review:          "name_review_flag" = 1
Temporally usable onset:           "temporal_use" = 'usable'
```

---

## Environment

Key dependencies and tested versions:

```yaml
name: geo
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11.9
  - geopandas=0.14.3
  - pandas=2.3.2
  - numpy=1.26.4
  - shapely=2.1.2
  - matplotlib=3.10.6
  - fiona=1.9.6
  - pyproj=3.7.2
  - osmnx
  - folium
  - requests
  - statsmodels
  - scipy
  - pip:
    - overturemaps
    - earthengine-api
    - geemap
    - pystac-client
    - planetary-computer
```

---

## Known Limitations

- **Footprint dating.** Overture/Microsoft footprints lack per-feature
  acquisition dates, so temporal attribution relies on CoStar and LODES rather
  than the footprints themselves.
- **OSM coverage** is sparse in some industrial zones; the name-matching
  classifier partially compensates but cannot cover all unnamed facilities.
- **NAIP roof detection** captures all large impervious surfaces, not just
  warehouses. Semantic classification in Step 3 filters these but may leave
  residual false positives.
- **Classification precision.** A name-based check flags ~74 facilities with
  non-warehouse names (civic / retail / residential) for manual review
  (`name_review_flag`); these may be genuine false positives or POI names that
  bled onto a real warehouse footprint via the buffered OSM-point join. A
  precision audit is recommended before citing the headline count as exact.
- **CoStar coverage.** CoStar covers ~34% of facilities and is subject to
  geocoding-precision artifacts; apparent coverage gaps at tight match radii
  collapse at wider radii and are handled by radius-sensitivity testing.
- **LODES vintage artifact.** The LODES7→LODES8 (2020) Census-block vintage
  change produces a spurious 2020 onset cluster (~2,030 facilities), excluded
  from temporal analysis via `temporal_use` but retained in the static
  inventory.
- **Temporal coverage.** Only ~56% of facilities have a reliable onset year;
  trajectory and anomaly results rest on this datable subset, not the full
  inventory.
- **RF outputs provisional.** `is_warehouse` / `warehouse_prob` reflect a
  random-forest model with a known feature-set mismatch and should be treated as
  provisional pending a retrain.

---

## Citation

If you use this code or methodology in your work, please cite:

```
Valencia-Cardenas, Maria C., & Jaller, Miguel. (2025). Empirical Assessment of
Land Use and Other Policy Impacts on Warehousing Location Choices in Southern
California. 2024-2025 NCST / Caltrans Research Grants.
GitHub: https://github.com/mariacvc/warehouse-inventory
```

---

## Contact

For questions about this project, please open a GitHub Issue or contact the research team.

---

## License

**Code** in this repository is released under the [choose a license — e.g., MIT]
license.

**Data are not covered by the code license and are not redistributed here.**
Open inputs (NAIP, Overture, Microsoft, OSM, TIGER, LODES, SCAG) remain under
their respective licenses; **CoStar data and CoStar-derived fields are
proprietary** and may not be redistributed without a valid CoStar license.
