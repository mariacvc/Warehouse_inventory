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
- **Institution:** Institute of Transportation Studies, UC Davis (ITS-Davis)
- **Status:** Research code — draft deliverable
---

## What this notebook does

`Warehouse_Inventory_Caltrans.ipynb` builds a reproducible facility inventory by
fusing four open building-footprint sources, classifying buildings, validating
against the Census County Business Patterns, and characterizing the spatial
pattern of the result.

1. **Study area** — TIGER/2018 county boundaries for the seven study counties.
2. **Footprint sources** — USDA NAIP (roof detection via Google Earth Engine),
   Microsoft US Building Footprints, Overture Maps, OpenStreetMap.
3. **Fusion & classification** — deduplication (IoU >= 0.3), multi-signal
   semantic classification (`confirmed` / `candidate` / `excluded`), an
   OSM-derived land-use spatial filter, and a functional subtype typology.
4. **Enrichment** — SCAG land-use layers, CoStar property records (construction
   year), and LEHD LODES employment (NAICS 48-49, 2003-2023).
5. **Validation** — ZIP-level comparison against Census CBP (NAICS 492 + 493).
6. **Spatial characterization** — Nearest Neighbor Index, quadrat counts,
   Ripley's K/L, weighted KDE, centrographic analysis (weighted mean center +
   standard deviational ellipse), and spatial/temporal anomaly detection.

## Study area

Seven counties: Los Angeles, Orange, Riverside, San Bernardino, Ventura,
Imperial, and **San Joaquin**. Footprint threshold: **>= 50,000 ft²**
(4,645.15 m²). Temporal frame: **2003-2023**.

## Three population counts (read this before citing numbers)

The notebook uses three related but distinct counts. They are not
interchangeable:

| Count | Meaning | Used for |
|-------|---------|----------|
| **Final inventory** | all included facilities (confirmed + high-confidence candidates, after manual review) | headline facility count, maps |
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
├── Inventory_documented.ipynb   # Main analysis notebook
│
├── data/                        # Downloaded raw data (not tracked in git)
│   ├── California.geojson.zip   # Microsoft building footprints (cached)
│   ├── la_buildings.geojson     # Overture buildings — Los Angeles
│   ├── or_buildings.geojson     # Overture buildings — Orange
│   ├── ri_buildings.geojson     # Overture buildings — Riverside
│   ├── sb_buildings.geojson     # Overture buildings — San Bernardino
│   └── ve_buildings.geojson     # Overture buildings — Ventura
│
├── Shapefiles/                  # Land use layers (not tracked in git)
│   ├── LA_osm_lulc_training.shp
│   ├── OR_osm_lulc_training.shp
│   ├── RI_osm_lulc_training.shp
│   ├── SB_osm_lulc_training.shp
│   └── VE_osm_lulc_training.shp
│
├── outputs/                     # Classified inventory files
│   └── warehouse_inventory_lu_filtered.gpkg
│
├── environment.yml              # Conda environment specification
└── README.md                    # This file
```

> **Note:** Raw data files (`.geojson`, `.zip`, `.shp`) are excluded from version control
> via `.gitignore` due to size. See the **Data Sources** section for download instructions.

---

## Methodology

The pipeline has four main stages:

```
Step 1: Define Study Area
        └─ TIGER/2018 county boundaries via Google Earth Engine

Step 2: Collect Building Footprints (four sources)
        ├─ USDA NAIP (roof detection via GEE)
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

## Data Sources

| Source | Description | Access |
|--------|-------------|--------|
| [USDA NAIP](https://developers.google.com/earth-engine/datasets/catalog/USDA_NAIP_DOQQ) | ~0.6 m aerial imagery (RGBN) | Google Earth Engine |
| [Overture Maps Buildings](https://docs.overturemaps.org/guides/buildings/) | Consolidated footprints + semantics | `overturemaps` CLI |
| [Microsoft Building Footprints](https://github.com/microsoft/USBuildingFootprints) | Computer vision roof polygons | Azure Blob (~5 GB) |
| [OpenStreetMap](https://wiki.openstreetmap.org/wiki/Key:building) | Named facilities, building tags | `osmnx` Python package |
| [TIGER/2018 Counties](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) | County boundaries | Google Earth Engine |

All sources are:
- Legally permissible for research use ✅
- Publicly accessible and reproducible ✅
- Scientifically defensible for peer review ✅
- Suitable for statewide scaling ✅

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

### 5. Set Your Path

In the notebook Setup cell, update:

```python
path = '/your/local/project/path/'
```

### 6. Run the Notebook

Execute cells sequentially. The Results section at the bottom runs the full pipeline.

---

## Output

The final output is a GeoPackage: `warehouse_inventory_lu_filtered.gpkg`

**Key columns:**

| Column | Type | Description |
|--------|------|-------------|
| `geometry` | Polygon | Building footprint (EPSG:4326) |
| `source` | str | Data source: `osm`, `overture`, or `ms` |
| `label` | str | `confirmed`, `candidate`, or `excluded` |
| `confidence_note` | str | Rule that triggered the label |
| `primary_name` | str | Facility name (from Overture/OSM) |
| `needs_review` | bool | `True` if label = `candidate` |
| `area_ft2` | float | Building footprint area in ft² |
| `compactness` | float | Shape metric (0–1; circles = 1.0) |
| `rectangularity` | float | Shape metric (0–1; rectangles = 1.0) |
| `elongation` | float | Shape metric (≥1; cross-docks > 2.0) |
| `lu_class` | str | Land use class from LULC shapefile |

### QGIS Quick-Start Filters

```
Confirmed warehouses:              "label" = 'confirmed'
Candidates needing review:         "needs_review" = 1
Excluded buildings:                "label" = 'excluded'
Confirmed DCs in exclusion zones:  "confidence_note" LIKE '%lu_conflict_kept%'
LU-confirmed warehouses:           "confidence_note" LIKE 'lu_confirm%'
Name-confirmed warehouses:         "confidence_note" LIKE 'name_keyword%'
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

- **Overture/Microsoft footprints** lack per-feature acquisition dates, so temporal
  attribution (2021 vs. 2022 vs. 2023 presence) requires cross-referencing with
  NAIP imagery or county orthos.
- **OSM coverage** is sparse in some industrial zones; the name-matching classifier
  partially compensates but cannot cover all unnamed facilities.
- **NAIP roof detection** captures all large impervious surfaces, not just warehouses.
  Semantic classification in Step 3 filters these but may have residual false positives.
- **County boundary edges** in the LULC shapefiles may create small unmatched zones.
  The overlap fallback in `assign_lu_to_buildings()` handles most of these cases.

---

## Citation

If you use this code or methodology in your work, please cite:

```
Valencia-CArdenas, Maria C., Jaller, Miguel. (2025). Empirical Assessment of Land 
Use and Other Policy Impacts on Warehousing Location Choices in Southern California.
2024-2025 NCST Caltrans Research Grants. GitHub: https://github.com/mariacvc/warehouse-inventory
```

---

## Contact

For questions about this project, please open a GitHub Issue or contact the research team.

---

## License
