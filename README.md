# Montpelier VT 2023 Flood — IN-CORE Input Preparation

This repository prepares the two hazard inputs required to run a community-level flood damage and population dislocation analysis in [IN-CORE](https://tools.in-core.org) for the July 2023 flood in Montpelier, Vermont:

1. **Flood depth raster** — a GeoTIFF of water depth above ground for the event peak
2. **Building inventory** — a Shapefile of structures with attributes compatible with IN-CORE's flood fragility framework

---

## Background

### The July 2023 Montpelier Flood

From July 9–12, 2023, an extreme rainfall event dropped 3–9 inches across Vermont in under 48 hours, producing catastrophic flooding along rivers statewide. Montpelier — Vermont's capital city — experienced some of the worst impacts. The Winooski River at Montpelier (USGS gage `04286000`) crested at **21.29 ft** on July 11, 2023, with a peak discharge of **23,100 cfs** — a record for that gage. The North Branch of the Winooski, which joins the mainstem directly in downtown Montpelier, also reached record stages. Downtown streets, businesses, and residences were inundated with several feet of water.

USGS crews documented the event by surveying 547 high-water marks across Vermont. A formal Scientific Investigations Report was subsequently published ([SIR 2025-5016](https://pubs.usgs.gov/publication/sir20255016/full)), and a data release of peak streamflow and flood frequency data is available on [ScienceBase](https://www.sciencebase.gov/catalog/item/664e23efd34e702fe8744b3c).

### IN-CORE

[IN-CORE](https://tools.in-core.org) (Interdependent Networked Community Resilience Modeling Environment) is an open-source platform developed at NCSA/UIUC for community resilience analysis. The Python library `pyIncore` enables programmatic access to IN-CORE's analysis modules. The flood damage workflow, illustrated in the [Lumberton NC Testbed](https://tools.in-core.org/doc/incore/notebooks/lumberton_testbed.html), requires:

- A **hazard dataset** — a raster of flood inundation depth in the study area
- A **building inventory** — a point or polygon layer of structures with occupancy type, foundation type, replacement cost, and first-floor elevation
- A **fragility mapping** — links building archetypes to flood fragility functions (provided by IN-CORE)

This repository produces the first two inputs for Montpelier using entirely free, publicly available data and runs entirely on macOS — no HEC-RAS installation or Windows required.

---

## Why Not HEC-RAS?

The standard approach to generating a flood depth raster is to run a hydrodynamic simulation in HEC-RAS (the Army Corps of Engineers hydraulic modeling software). This is exactly what was done for the Lumberton testbed: HEC-RAS was run with a high-resolution DEM and USGS hydrographs to produce a 2D inundation depth grid.

However, HEC-RAS:
- **Requires Windows** — it uses a COM (Component Object Model) interface that is not available on macOS or Linux
- **Requires significant modeling experience** — setting up geometry, boundary conditions, and a 2D mesh is a multi-week effort for a new user

Python libraries like `ras-commander`, `raspy`, and `rascontrol` automate HEC-RAS but still require a Windows machine with HEC-RAS installed. They are controllers, not standalone simulators.

For this project, we bypass HEC-RAS entirely by using pre-computed flood inundation maps published by NOAA and USGS.

---

## Data Sources

### Flood Depth Raster — NOAA Flood Inundation Mapping (FIM) Library

NOAA's National Weather Service operates a **Flood Inundation Mapping** program that publishes pre-computed inundation depth grids for hundreds of river reaches across the US, keyed to the AHPS (Advanced Hydrologic Prediction Service) stream gauges.

The Winooski River at Montpelier is covered by the AHPS gauge **MONV1**, and NOAA has published a FIM library for this location:

- **Download:** https://water.noaa.gov/resources/downloads/fim/btv/monv1/
- **Format:** Esri GRID rasters (`.adf` files), one per water surface elevation increment
- **Coverage:** Water surface elevations from 515.2 ft to 528.4 ft NAVD88
- **Resolution:** ~2.44 meters per pixel
- **CRS:** EPSG:3857 (WGS 1984 Web Mercator Auxiliary Sphere)

Each raster in the library represents the depth of flooding (in feet above ground) across the study area if the water surface at the MONV1 gauge reaches that elevation.

**Selecting the right grid for July 2023:**

The USGS gage datum (elevation of the gage zero) for site `04286000` is **499.87 ft NAVD88** (queried from the USGS site service, `alt_va` field). At peak stage of **21.29 ft**, the water surface elevation was:

```
WSE = gage_height + gage_datum = 21.29 + 499.87 = 521.16 ft NAVD88
```

The closest pre-computed grid in the NOAA FIM library is **`elev_521_6`** (WSE = 521.6 ft NAVD88), which is 0.44 ft above the actual peak — a conservative slight overestimate of inundation extent and depth.

### Building Inventory — National Structure Inventory (NSI)

The [National Structure Inventory](https://www.hec.usace.army.mil/confluence/nsi) (NSI), maintained by USACE, provides a nationwide database of structure locations and attributes including:

- Occupancy type (HAZUS classification: RES1, COM1, etc.)
- Number of stories
- Foundation type
- Ground elevation (ft NAVD88)
- First floor height above ground
- Structural replacement cost
- Contents value
- Square footage

The NSI is accessible via a free REST API with no authentication required. The GET endpoint returns a 500 error for some bounding boxes, but the **POST endpoint with a GeoJSON polygon body** is reliable:

```
POST https://nsi.sec.usace.army.mil/nsiapi/structures?fmt=fc
Body: GeoJSON Feature with polygon geometry
```

---

## Environment Setup

This project uses a dedicated conda environment to avoid NumPy version conflicts with the Anaconda base environment (which has NumPy 2.x, incompatible with some dependencies).

```bash
# Create environment with Python 3.10 and NumPy 1.x
/opt/anaconda3/bin/conda create -n hecras python=3.10 "numpy<2" -y

# Install dependencies
/opt/anaconda3/envs/hecras/bin/pip install \
    dataretrieval geopandas rasterio boto3 s3fs requests scipy xarray zarr
```

**Why not `fimserve`?** The `fimserve` Python package (which wraps NOAA's HAND-FIM framework) was initially considered but proved uninstallable due to an unresolvable dependency graph (`teehr==0.5.0` has deeply conflicting transitive dependencies). The NOAA HAND-FIM S3 bucket (`noaa-nws-owp-fim`) also requires ESIP credentials and is not fully public. The NOAA FIM library approach used here is simpler, more reliable, and produces directly observed-event-calibrated results.

---

## Outputs

Both files are saved to `./outputs/`:

### `montpelier_flood_depth_2023.tif`

| Property | Value |
|----------|-------|
| Format | GeoTIFF |
| CRS | EPSG:3857 (Web Mercator) |
| Resolution | ~2.44 m/pixel |
| Units | Feet above ground surface |
| Extent | Downtown Montpelier floodplain (`-72.590, 44.250, -72.550, 44.270`) |
| Source grid | NOAA FIM `elev_521_6` (WSE 521.6 ft NAVD88) |
| Max depth | 23.78 ft |
| Mean wet-cell depth | 6.77 ft |
| Inundated cells | 202,219 |

### `montpelier_building_inventory/`

| Property | Value |
|----------|-------|
| Format | ESRI Shapefile |
| CRS | EPSG:4326 (WGS84) |
| Source | NSI API, POST query |
| Extent | Same downtown bounding box |
| Total structures | 2,131 |
| Schema | Matches IN-CORE Lumberton testbed (`guid`, `strctid`, `g_elev`, `ffe_elev`, `no_stories`, `occ_type`, `archetype`, `repl_cst`, `sq_foot`) |

**Archetype distribution** (Nofal & van de Lindt 2020, archetypes 1–15):

| Archetype | Description | Count |
|-----------|-------------|-------|
| 1 | 1-story residential, slab/crawl | 420 |
| 3 | 2-story residential | 694 |
| 6 | Mobile home | 1 |
| 7 | Small commercial | 116 |
| 8 | Wholesale/large commercial | 32 |
| 9 | Office/professional | 119 |
| 10 | Large office/institutional | 81 |
| 11 | Industrial/warehouse | 41 |
| 12 | Multifamily 3+ units | 345 |
| 13 | Religious/recreational | 89 |
| 14 | School | 12 |
| 15 | Essential facility | 10 |
| Unmapped | RES1-3SNB, RES1-3SWB (3-story SF) | 171 |

---

## Using These Outputs in pyIncore

The outputs slot directly into the Lumberton testbed workflow. Replace the Lumberton hazard and building IDs with local file references:

```python
from pyincore import IncoreClient, Dataset, FragilityService, MappingSet, DataService
from pyincore.analyses.buildingdamage import BuildingDamage

client = IncoreClient()

# Load hazard from local file
flood_dataset = Dataset.from_file(
    'outputs/montpelier_flood_depth_2023.tif',
    data_type='incore:floodRaster'
)

# Load building inventory from local file
bldg_dataset = Dataset.from_file(
    'outputs/montpelier_building_inventory/montpelier_building_inventory.shp',
    data_type='ergo:buildingInventoryVer7'
)

# Fragility mapping — same as Lumberton (archetypes 1–15 are identical)
mapping_id = "602f3cf981bd2c09ad8f4f9d"
fragility_service = FragilityService(client)
mapping_set = MappingSet(fragility_service.get_mapping(mapping_id))

# Run building damage
bldg_dmg = BuildingDamage(client)
bldg_dmg.set_input_dataset("buildings", bldg_dataset)
bldg_dmg.set_input_dataset("dfr3_mapping_set", mapping_set)
bldg_dmg.set_parameter("hazard_type", "flood")
bldg_dmg.set_parameter("result_name", "montpelier_flood_dmg")
bldg_dmg.set_parameter("fragility_key", "Lumberton Flood Building Fragility ID Code")
bldg_dmg.set_parameter("num_cpu", 4)
bldg_dmg.run_analysis()
```

For population dislocation, follow the Lumberton notebook using Washington County, VT Census block group data (FIPS `50023`) in place of Robeson County, NC (`37155`).

---

## Known Limitations

| Issue | Detail |
|-------|--------|
| FIM grid is 0.44 ft above actual peak | `elev_521_6` (521.6 ft) vs actual 521.16 ft — slightly overestimates inundation extent |
| FIM library covers Winooski mainstem only | Flooding from the North Branch tributary and surface runoff in side streets may be underrepresented |
| Archetype crosswalk is approximate | NSI HAZUS codes are mapped to IN-CORE archetypes based on building type descriptions; field validation or parcel permit data would improve accuracy |
| 171 buildings unmapped | `RES1-3SNB` and `RES1-3SWB` (3-story single-family) not in the original crosswalk; should be mapped to archetype 3 |
| `ground_elv` in NSI is modeled | NSI ground elevations come from the 3DEP DEM, not field survey; individual building FFE may differ |
| Downtown bounding box only | This covers the central Montpelier floodplain; expand `BBOX` in the notebook to include surrounding neighborhoods |
| No North Branch model | The North Branch Winooski joins downtown from the north; a separate FIM library would be needed for that reach if it exists |

---

## File Structure

```
hecras/
├── README.md                          # this file
├── montpelier_flood_incore.ipynb      # full workflow notebook
├── outputs/
│   ├── montpelier_flood_depth_2023.tif          # flood depth raster → IN-CORE hazard
│   └── montpelier_building_inventory/           # building shapefile → IN-CORE exposure
│       ├── montpelier_building_inventory.shp
│       ├── montpelier_building_inventory.dbf
│       ├── montpelier_building_inventory.prj
│       └── montpelier_building_inventory.shx
├── 02010003/                          # (if HAND approach used) NOAA HAND data cache
└── noaa_fim/
    └── shapefiles/shp/ahps/inundation/monv1/
        ├── depth_grids/               # NOAA FIM depth rasters (elev_515_2 – elev_528_4)
        └── polygons/                  # NOAA FIM inundation polygons
```

---

## References

- van de Lindt et al. (2020). Community resilience-focused technical investigation of the 2016 Lumberton, NC, flood. *Natural Hazards Review*. https://doi.org/10.1061/(ASCE)NH.1527-6996.0000387
- Nofal, O.M. & van de Lindt, J.W. (2020). Probabilistic flood loss assessment at the community scale. *ASCE-ASME Journal of Risk and Uncertainty in Engineering Systems*. https://doi.org/10.1061/AJRUA6.0001060
- USGS (2025). Flood of July 2023 in Vermont. Scientific Investigations Report 2025-5016. https://pubs.usgs.gov/publication/sir20255016/full
- USGS ScienceBase data release: https://www.sciencebase.gov/catalog/item/664e23efd34e702fe8744b3c
- NOAA MONV1 FIM library: https://water.noaa.gov/resources/downloads/fim/btv/monv1/
- USGS gage 04286000 (Winooski River at Montpelier): https://waterdata.usgs.gov/monitoring-location/USGS-04286000/
- NSI API documentation: https://www.hec.usace.army.mil/confluence/nsi
- IN-CORE Lumberton testbed: https://tools.in-core.org/doc/incore/notebooks/lumberton_testbed.html
