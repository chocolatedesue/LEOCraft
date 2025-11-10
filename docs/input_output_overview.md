# LEOCraft Inputs & Outputs

This note condenses the levers you must set before running a simulation plus the artifacts LEOCraft can emit afterwards. Use it as a checklist when wiring new constellations, traffic scenarios, or visualization flows.

---

## Required Inputs

### 1. Runtime Prerequisites
- **Conda toolchain** – create/activate the `leocraft` env with `tools/environment.yml` (or `environment_macOS.yml`) and export `PYTHONPATH=$(pwd)` so in-place imports resolve. See `README.md` (“Setup the Simulation Environment”).
- **Gurobi license** – download `grbgetkey`, request an academic license, and store `gurobi.lic` in `$HOME`; throughput LPs fail without it. Instructions live in `README.md` (“Get the Academic License for Gurobi Optimizer”).

### 2. Ground Segment & Traffic Data
- **Ground stations** – pick one of the CSVs exposed via `GroundStationAtCities` (e.g., `dataset/ground_stations/cities_sorted_by_estimated_2025_pop_top_100.csv`). `LEOCraft/user_terminals/ground_station.py` turns those rows into `TerminalCoordinates`.
- **Traffic matrices** – throughput runs need a JSON from `InternetTrafficAcrossCities` (e.g., `dataset/traffic_metrics/population_only_tm_Gbps_100.json`) or aviation files under `dataset/air_traffic/`. These feed `Throughput._process_traffic_metrics`.
- **Optional aviation data** – `FlightOnAir` datasets (`dataset/aircraft/*.csv`) substitute aircraft nodes for select ground stations in aviation constellation scenarios.

### 3. Constellation Definition
- **Shell topology parameters** – each `PlusGridShell` (or derived topology) requires `id`, `orbits`, `sat_per_orbit`, `altitude_m` or `altitude_pattern_m`, `inclination_degree`, `angle_of_elevation_degree`, and `phase_offset`. These values drive TLE synthesis in `LEOCraft/satellite_topology/plus_grid_shell.py`.
- **Constellation assembly** – instantiate `LEOConstellation`, call `add_ground_stations(GroundStation(...))`, `add_shells(shell)`, optional `set_time(...)`, and `set_loss_model(FSPL(...))` before calling `build()`/`create_network_graph()`/`generate_routes()`.
- **Loss model knobs** – `FSPL` expects frequency, transmit power, bandwidth, and G/T ratio; call `set_Tx_antenna_gain()` and `set_Rx_antenna_gain()` with either gains or dish diameters. Without a loss model, `Constellation` falls back to the default `GSL_CAPACITY`.

### 4. Visualization & Analysis Selections
- Declare which satellites, coverages, routes, or links to render through `SatView2D`/`SatView3D`/`SatRawView3D` helpers (`add_all_satellites`, `add_coverages`, `add_routes`, etc.).
- Choose performance modules as needed: `Throughput`, `Coverage`, `Stretch` (and aviation variants) share the constellation instance and depend on the above datasets and shells.

---

## Generated Outputs

### 1. Console Diagnostics
- Build/compute phases log timing, progression, throughput, stretch, and coverage metrics thanks to `ProcessingLog`. Expect banners like `[LEOConstellation] Building shells...` or `[Throughput] Throughput: 6144.55 Gbps` (see `README.md` examples).

### 2. Shell-Level Files
- `shell.export_satellites(prefix)` → `<PlusGridShell_*>.tle` triples for every satellite in the shell.
- `shell.export_isls(prefix)` → `<PlusGridShell_*>.isls.csv` capturing all inter-satellite links as `(sat_a, sat_b)`.

### 3. Constellation-Level Exports
Located under `/<prefix>/<time_delta>/` (created via `Constellation._create_export_dir`):
- `leo_con.export_gsls()` – JSON map of every ground station to its in-range satellites plus distances.
- `leo_con.export_routes()` – JSON of K-shortest routes per GS pair.
- `leo_con.export_no_path_found()` / `export_k_path_not_found()` – TXT lists for missing or incomplete flows.
- `leo_con.ground_stations.export()` – CSV dump of all `TerminalCoordinates`.

### 4. Performance Artifacts
- `Throughput` – `path_selection.json` (LP-selected routes), `<Throughput>.lp` (Gurobi model), and console printouts of throughput and accommodated-flow percentages.
- `Stretch` – `stretch.csv` containing per-flow stretch metrics plus medians logged per route class.
- `Coverage` – logs number of out-of-coverage ground stations and coverage metric (no separate file).

### 5. Visualizations
- `SatView2D.export_html()` – Folium map HTML showing satellites, coverage cones, GSLs, and ISLs.
- `SatView3D.export_html()` / `export_png()` (via `SatRawView3D`) – Plotly 3D scatter exports for orbit visualizations or static figures for `docs/visuals/`.

### 6. Example Pipeline
`examples/example_starlink.py` wires everything together:
1. Configure FSPL, load TOP_100 ground stations, and add three PlusGrid shells.
2. `set_time()`, `set_loss_model()`, `build()`, `create_network_graph()`, `generate_routes()`.
3. Run `Throughput`, `Coverage`, `Stretch`.
4. Export GSLS/routes/no-path/k-path, each shell’s TLE/ISL, ground-station CSV, throughput artifacts, and stretch dataset into `./Starlink/`.

Use that script (or a derivative) as a template for new constellations; swap in alternate datasets or shells, then consult the sections above to ensure every required input is defined and every desired output is explicitly exported.
