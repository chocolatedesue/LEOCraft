# Flow-Level Simulation Design

## Architecture Snapshot
LEOCraft treats a constellation as a graph whose nodes are ground stations and satellites, and whose edges are ground-to-satellite links (GSLs) plus inter-satellite links (ISLs). `Constellation` (`LEOCraft/constellations/constellation.py`) encapsulates the common build steps—loading `GroundStation` data, instantiating satellite shells such as `PlusGridShell`, computing visibility windows, and attaching link capacity metadata so downstream performance modules can consume a consistent topology.

## Data Inputs & Preparation
Physical sites and traffic are versioned under `dataset/`. `GroundStationAtCities` and its CSVs define terminal coordinates for different deployment scales (top-100, top-1000, capitals, etc.). Flow demand matrices (e.g., `dataset/traffic_metrics/population_only_tm_Gbps_100.json`) store bidirectional throughput targets inferred via a gravity model. Aviation-focused datasets reuse the same pattern with `FlightOnAir` and `InternetTrafficOnAir`. When a scenario is assembled (see `examples/example_starlink.py`), the consumer selects the ground-station CSV, traffic matrix, shell mix, observation time, and optional FSPL-based loss model that controls dynamic GSL capacities.

## Build & Routing Workflow
1. **Shell Construction** – Each `PlusGridShell` (or other `LEOSatelliteTopology`) generates satellite states and ISL pairs for the requested time delta. The constellation enforces `shell.id == index` so multi-shell fleets remain ordered.
2. **Visibility Scan** – `Constellation.build()` parallelizes the per-station/per-shell visibility search, recording for each satellite which ground stations it covers and the range in meters (`sat_coverage`, `gsls`). If a loss model is set, GSL capacities are derived from `FSPL.data_rate_bps(distance, fanout)`; otherwise they default to evenly split static caps.
3. **Graph Assembly** – `create_network_graph()` loads all satellites and ISLs into a NetworkX graph, then connects ground stations on demand. Each edge stores both `weight` (distance) and `capacity` attributes, enabling later constraints on stretch and throughput.
4. **K-Shortest Routes** – `LEOConstellation.generate_routes()` iterates over every ordered ground-station pair, attaches the two endpoints to the graph, and invokes `k_shortest_paths` to retrieve `Constellation.k` (default 20) simple routes. The method records every route’s hop sequence and annotates each traversed edge in `link_load`, which is the dependency graph for the LP stage.

## Flow-Level Optimization
`ThroughputLP` (`LEOCraft/performance/throughput_LP.py`) bridges the discrete routing catalog and continuous traffic demands:
- `_process_traffic_metrics()` merges the JSON’s bidirectional entries so each flow key `G-X_G-Y` represents total symmetric demand. The basic implementation lives in `performance/basic/throughput.py`.
- `FlowClassifier` (`performance/route_classifier`) buckets flows by orientation (north–south, east–west, NESW, high/low geodesic) to power downstream reporting.
- During `build()`, the LP connects *all* ground stations simultaneously to expose every route, then `compute()` forms a multi-commodity linear program: decision variables `R[flow, path]` ∈ [0,1], objective maximize ∑ demand × selection, constraints enforce ≤1 route per flow and cumulative load ≤ edge capacity for every `(node_a, node_b)` stored in `link_load`.
- After Gurobi solves the model, helper methods extract the chosen route fractions, overall throughput (Gbps), percent of accommodated flows, and selection ratios per flow class. Artifacts such as `path_selection.json` and the `.lp` file can be exported for audit or reuse.

## Batch Simulation Harness
`Simulator` (`LEOCraft/simulator/simulator.py`) is the orchestration layer for sweeps. You enqueue prepared `Constellation` objects via `add_constellation()`, then run either `simulate_in_serial()` or `simulate_in_parallel()` to execute build → route → metrics pipelines for each job. The abstract `_simulate()` in concrete simulator classes typically instantiates `Throughput`, `Coverage`, and `Stretch`, captures their metrics, and streams them into a CSV log per run (time delta, shell parameters, throughput, coverage, stretch stats, etc.). This structure makes it easy to compare constellation variants or time snapshots without rewriting plumbing.

## Extending the Flow-Level Stack
- **New Traffic Models**: Drop additional JSON matrices under `dataset/traffic_metrics/` and implement a `_process_traffic_metrics()` override if flows require custom aggregation (e.g., asymmetrical demand).
- **Custom Topologies**: Create another `LEOSatelliteTopology` subclass to encode shell geometry, ISL plan, or dynamic ISL range enforcement, then register it through `Constellation.add_shells()`.
- **Alternative Performance Studies**: Reuse the routing catalog and `link_load` with bespoke optimizers or metrics. For example, a different LP could penalize hop count or prioritize specific flow classes while keeping the existing build infrastructure intact.
