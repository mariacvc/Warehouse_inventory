# Data Dictionary — Southern California Warehouse Inventory

One row per facility; 8,826 large warehouse/distribution facilities (>=50,000 ft2), six SCAG counties.
Companion to `warehouse_inventory_clean.csv`. Spatial geometry is in the `.gpkg`; this CSV carries centroids.

| Column | Definition | Values / units | Source | Non-null |
|---|---|---|---|---|
| `facility_id` | Stable surrogate primary key assigned by this export (SCAG-WH-#####). | string | this export | 100% |
| `overture_id` | Original Overture building ID; null for Microsoft/OSM-sourced footprints. | string | Overture | 61% |
| `facility_name` | Facility/operator name where available; parsed from OSM/Overture tags. Mostly null — inventory is footprint-derived, not name-derived. | string | OSM/Overture | 7% |
| `name_review_flag` | TRUE = the name matches non-warehouse keywords (civic/retail/residential) with no warehouse term; candidate for manual/imagery review (possible false positive or name-bleed). FALSE otherwise. | bool | this export | 100% |
| `county` | SCAG county (LA, OR, RI, SB, VE, IMP). | categorical | SCAG / nearest-neighbor | 100% |
| `city` | Municipality. | string | SCAG / nearest-neighbor | 100% |
| `county_source` | 'original' = assigned in pipeline; 'nearest_neighbor' = backfilled here from the nearest labeled facility (validate against boundary polygons). | categorical | this export | 100% |
| `centroid_lon` | Centroid longitude, WGS84 (EPSG:4326), computed via EPSG:3310. | degrees | this export | 100% |
| `centroid_lat` | Centroid latitude, WGS84 (EPSG:4326). | degrees | this export | 100% |
| `area_ft2` | Building footprint area. | sq ft | footprint geometry | 100% |
| `label` | Inclusion class: 'confirmed' (full criteria) or 'candidate' (high-confidence, promoted). | categorical | classifier | 100% |
| `subclass` | Functional subtype for confirmed buildings: FI-IND, U-DF, R-DC, R-RL, ambiguous. Candidates = 'candidate (unsubtyped)'. | categorical | classifier | 100% |
| `confidence_score` | Additive rule-based score 0-100 (label + operator/shape notes + NAIP signals). | 0-100 | classifier | 100% |
| `confidence_tier` | Binned confidence (Low/Review/High). | categorical | classifier | 100% |
| `is_warehouse` | Random-forest binary prediction (1=warehouse). Provisional — RF feature-set mismatch noted in QA. | 0/1 | RF model | 100% |
| `warehouse_prob` | Random-forest warehouse probability. | 0-1 | RF model | 100% |
| `source` | Footprint provenance: overture / ms (Microsoft) / osm. | categorical | fusion | 100% |
| `scag_land_use_class` | SCAG parcel land-use class at the site (e.g., warehouse, industrial, non_industrial). | categorical | SCAG | 97% |
| `APN` | Assessor parcel number (public). | string | county assessor | 97% |
| `costar_match_confidence` | Quality of CoStar record match: high / medium / weak / none. | categorical | CoStar matcher | 100% |
| `intensity_tier` | Operational-intensity tier (populated where CoStar attributes available). | categorical | CoStar | 34% |
| `is_high_cube` | High-cube facility flag (CoStar-derived where available). | bool | CoStar | 34% |
| `is_likely_crossdock` | Likely cross-dock flag (shape/CoStar-derived). | bool | derived | 34% |
| `ops_start_year` | RESOLVED start year used in all temporal analysis (hierarchy: CoStar > OSM > LODES). | year | derived (canonical) | 79% |
| `year_source` | Which input fed ops_start_year: costar / osm / lodes / unknown. | categorical | derived | 100% |
| `year_confidence` | Confidence in ops_start_year (high/medium/low). | categorical | derived | 100% |
| `lodes_onset_year` | INPUT: first year LODES shows logistics employment at the block. Differs from year_built by design (employment vs construction). | year | LODES | 72% |
| `year_built` | INPUT: CoStar construction year (~34% coverage). | year | CoStar | 34% |
| `temporal_use` | Analysis validity of the onset: 'usable', 'lodes8_artifact' (false 2020 onset, excluded), 'lodes7_truncation' (left-truncated, excluded), 'no_onset'. | categorical | this export | 100% |
| `suspect_stale_year` | Flagged as possible demolish-replace (old CoStar year vs recent onset). QA flag only. | bool | derived | 100% |

### Notes
- **Which year to use:** `ops_start_year` is canonical; `year_built` and `lodes_onset_year` are its two inputs and disagree by design. `year_source` records which input was used per row.
- **Temporal subset:** for trend/centrographic work, filter `temporal_use == 'usable'`; the LODES8 2020 artifact and 2003 left-truncation are excluded there.
- **`name_review_flag == True` (n=74):** names suggest non-warehouse use; review before citing as warehouses (possible false positive or name-bleed from the OSM-point join).
- **`county_source == 'nearest_neighbor'` (n=276):** county/city backfilled from the nearest labeled facility; validate against boundary polygons for the final release.
- **Candidates:** `label == 'candidate'` (1,125) are high-confidence but not functionally subtyped (`subclass == 'candidate (unsubtyped)'`).