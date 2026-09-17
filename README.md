

# cyclone-data

Offline data and ML pipeline for the **Cyclone AI Command Center**. This repo produces every static artifact — satellite layers, terrain, infrastructure, cyclone scenarios, and calibrated ML models — that `cyclone-api` reads at runtime. Nothing in here is called live during a demo; it runs ahead of time and commits its outputs.

## What this repo does

- Pulls and processes satellite/terrain data via Google Earth Engine (land cover, built-up extent, water occurrence, elevation)
- Extracts critical infrastructure (hospitals, shelters, substations, roads, bridges, schools) from OpenStreetMap
- Prepares historical cyclone tracks from IBTrACS as demo scenarios
- Fits a HAND-based flood model and calibrates its threshold against a real event (Cyclone Hudhud, 2014)
- Trains a lightweight infrastructure damage-probability model
- Outputs versioned GeoJSON, raster, and model files consumed by `cyclone-api`

## Repo structure

```
cyclone-data/
├── gee/
│   ├── export_landcover.ipynb
│   ├── export_dem.ipynb
│   └── export_water_occurrence.ipynb
├── osm/
│   ├── fetch_infrastructure.py       # Overpass queries: hospitals, shelters, substations, roads
│   └── clean_infrastructure.py
├── ibtracs/
│   ├── fetch_tracks.py               # IBTrACS pull for Hudhud, Phailin, + 1 weaker storm
│   └── scenarios/                    # cleaned per-cyclone GeoJSON tracks
├── ml/
│   ├── flood_calibration/
│   │   ├── fit_hand_threshold.py
│   │   ├── validation_hudhud.ipynb
│   │   └── calibrated_params.json    # OUTPUT
│   ├── damage_model/
│   │   ├── train_damage_model.py
│   │   ├── features.py
│   │   ├── damage_model.pkl          # OUTPUT
│   │   └── feature_importance.json   # OUTPUT
│   └── track_interpolation/
│       └── spline_interpolate.py
├── outputs/                          # final GeoJSON/raster artifacts for cyclone-api
├── requirements.txt
└── README.md
```

## Setup

```bash
git clone <repo-url>
cd cyclone-data
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

You'll need:
- A Google Earth Engine service account (or personal GEE auth) for the `gee/` notebooks
- Internet access for OSM Overpass and IBTrACS pulls (both free, no key required)

## Running the pipeline

Run once, in order, whenever data needs refreshing:

```bash
# 1. Terrain + satellite layers
jupyter nbconvert --execute gee/export_dem.ipynb
jupyter nbconvert --execute gee/export_landcover.ipynb

# 2. Infrastructure
python osm/fetch_infrastructure.py --region kakinada
python osm/clean_infrastructure.py

# 3. Cyclone scenarios
python ibtracs/fetch_tracks.py --storms hudhud,phailin

# 4. Calibrate flood model against Hudhud
python ml/flood_calibration/fit_hand_threshold.py

# 5. Train damage model
python ml/damage_model/train_damage_model.py
```

All outputs land in `outputs/` and should be committed (or pushed to a shared bucket/release if file sizes get large — use Git LFS for anything over ~50MB).

## Study area

Scoped to a single coastal district — **Kakinada, Andhra Pradesh** — to keep the MVP demo-able in a week. Bounding box and CRS are defined in `osm/fetch_infrastructure.py`.

## Validation

`ml/flood_calibration/validation_hudhud.ipynb` scores predicted flood extent against observed inundation from Cyclone Hudhud (Oct 2014) using IoU. This is the credibility check referenced in the main project pitch — see the output metric before presenting.

## Consumed by

- [`cyclone-api`](../cyclone-api) — reads everything in `outputs/`, including `calibrated_params.json` and `damage_model.pkl`

## Disclaimer

All outputs are decision-support estimates for demo purposes, not operational forecasts. Real disaster response should rely on official IMD/NDMA warnings.
