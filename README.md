# GeoDispatch

**Adaptive Emergency Coverage Optimizer** — a spatial dispatch engine for emergency facilities (hospitals, fire stations) built for the Advanced Data Structures (ADS) course at Vishwakarma Institute of Technology, Pune.

GeoDispatch answers three questions for a city's emergency network in real time:

- Given an incident location, **which facility responds** (nearest, or top-*k* nearest)?
- If a facility is optimally repositioned, **how does coverage improve** (Lloyd's relaxation)?
- Which parts of the city are **under-served** right now (Voronoi coverage map)?

All core geometry runs in C11; a stdlib-only Python server exposes it over HTTP; a Leaflet-based frontend visualizes it on a live map of Pune.

## Architecture

```
frontend/ (HTML/CSS/JS, Leaflet map)
      │  fetch()
      ▼
python/server.py  (stdlib http.server — routing, JSON, state only)
      │  subprocess, stdin/stdout JSON
      ▼
geodispatch(.exe)  (C11 — all data structures & algorithms)
      │
      ▼
data/pune_facilities.json  (644 real facilities: 634 hospitals, 10 fire stations)
```

Python never computes geometry — it only loads data, tracks facility on/off state, and pipes queries to the C binary. All spatial logic (KD-tree, Voronoi, Lloyd's relaxation) lives in C.

## Core algorithms (`src/`)

| File | Structure / Algorithm | Purpose |
|---|---|---|
| `kd.c` | KD-tree | Nearest-neighbor and k-NN queries over facility points |
| `kd_dynamic.c` | Lazy deletion + rebalancing | Keep the tree correct as facilities go offline/online without a full rebuild each time |
| `voronoi.c` | Half-plane intersection + Sutherland–Hodgman clipping | Compute each facility's Voronoi cell (its service region) |
| `algo.c` | Lloyd's relaxation | Iteratively move facilities to their cell centroid to optimize coverage |
| `bench.c` | KD-tree vs. brute force | Benchmarks nearest-neighbor lookup at scale |

`src/main.c` wraps these into a single CLI (`geodispatch`) with four commands: `nearest`, `knn`, `optimise`, `coverage`. It reads facilities from stdin and prints JSON to stdout, so the Python layer can call it as a subprocess per request.

### Why a KD-tree instead of brute force

| n (facilities) | KD-tree (µs) | Brute force (µs) |
|---:|---:|---:|
| 1,000 | 0.09 | 7.0 |
| 10,000 | 0.15 | 70.0 |
| 50,000 | 0.25 | 350.0 |

Full results in [`data/benchmark_results.csv`](data/benchmark_results.csv) and [`data/benchmark_report.pdf`](data/benchmark_report.pdf).

## Coordinate system

All geometry is done in a local equirectangular projection centered on Pune (`lat0 = 18.5204, lon0 = 73.8567`), converting lat/lon to flat x/y metres and back. This keeps the C-side geometry pure Euclidean, with lat/lon conversion only at the API boundary.

## API (`python/server.py`)

| Method | Route | Description |
|---|---|---|
| GET | `/facilities` | All facilities with current state |
| GET | `/facility-states` | Online/offline/overloaded state map |
| GET | `/live-facilities` | IDs currently online |
| GET | `/coverage-map` | Voronoi cells as GeoJSON, flagged if under-served |
| POST | `/query-nearest` | Nearest facility to a `{lat, lon}` |
| POST | `/query-knn` | k nearest facilities to a `{lat, lon}` |
| POST | `/optimise` | Run Lloyd's relaxation, returns facility movements |
| POST | `/set-state` | Toggle a facility online/offline/overloaded |
| POST | `/add-facility` | Add a new facility at `{lat, lon}` |
| DELETE | `/remove-facility/{id}` | Remove a facility |
| POST | `/reset` | Reset facilities/state to the original dataset |

## Frontend (`frontend/`)

A Leaflet map with three modes:

- **Query** — click the map to find the nearest facility, or run a k-NN search.
- **Edit** — place or remove facilities live.
- **Optimise** — run Lloyd's relaxation and view the resulting coverage map / benchmark report.

## Running it

Requires `gcc` and Python 3 (no pip packages).

```bash
# 1. Build the C engine
make            # or build.bat on Windows

# 2. Start the server
python python/server.py

# 3. Open the app
# http://localhost:8000
```

Other Make targets: `make run-bench` (KD-tree benchmark), `make run-test` (unit tests), `make clean`.

## Project layout

```
src/            KD-tree, Voronoi, Lloyd's relaxation, CLI entry point, tests, benchmark
python/         HTTP server (routing + state, zero external dependencies)
frontend/       Leaflet map UI (HTML/CSS/JS)
data/           Pune facility dataset, benchmark results
Makefile / build.bat   Build the C engine on Linux/macOS or Windows
```

## Team

A 5-person team project for the ADS course, B.Tech CSE (AI & ML), Batch 2027 — see [`GeoDispatch_Context_Aware_Doc.pdf`](GeoDispatch_Context_Aware_Doc.pdf) for the full technical specification.
