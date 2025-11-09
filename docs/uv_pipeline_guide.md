# UV Workflow Guide

This guide explains how to run LEOCraft with [Astral's `uv`](https://github.com/astral-sh/uv) instead of Conda, including environment bootstrap, dependency installation, and example pipeline commands.

## 1. Prerequisites
- Python 3.12 compatible toolchain (LEOCraft targets 3.12 per `tools/environment.yml`).
- Gurobi academic or commercial license (`gurobi.lic`) placed in your home directory so throughput LPs can run.
- `uv` installed globally:
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
  After installation, ensure `~/.cargo/bin` (Linux/macOS) is on your `PATH`.

## 2. Create and Activate a UV Environment
```bash
cd /path/to/LEOCraft
uv venv .venv               # create project-local virtual environment
source .venv/bin/activate   # macOS/Linux shell
# On Windows (PowerShell): .\.venv\Scripts\Activate.ps1
```

## 3. Install Python Dependencies via UV
We maintain a pip-compatible list at `tools/requirements-uv.txt` that mirrors the Conda environment's Python packages.
```bash
uv pip install -r tools/requirements-uv.txt
```
Notes:
- UV caches wheels globally, so repeated installs are fast.
- Some system libraries (e.g., `libgomp`) still come from your OS package manager; install them with `apt`, `brew`, etc., if pip wheels complain.

## 4. Configure Project Environment
```bash
export PYTHONPATH=$(pwd)
# Optional: pin Gurobi license path if not in $HOME
export GRB_LICENSE_FILE=${HOME}/gurobi.lic
```
If you plan to run multiple shells in parallel, also set `OMP_NUM_THREADS` or `UV_LINK_MODE=copy` as needed for your platform.

## 5. Run the Standard Pipeline with UV
With dependencies installed and the venv active, you can launch scripts using either `python` (inside the venv) or `uv run`:
```bash
uv run python examples/example_starlink.py
```
This command executes the full Starlink example: build constellation → compute routes → run throughput/coverage/stretch → export artifacts under `./Starlink/`.

### Inputs for a Typical Run
- **Constellation definition**: shells you instantiate in the script (e.g., orbit count, altitude, inclination, phase offset for each `PlusGridShell`).
- **Ground-station dataset**: a CSV path exposed via `GroundStationAtCities` (TOP_100, TOP_1000, COUNTRY_CAPITALS, etc.).
- **Traffic matrix**: JSON file specified through `InternetTrafficAcrossCities` or aviation datasets; determines flow demand between stations.
- **Optional physics knobs**: FSPL parameters (frequency, power, antenna gain), simulation time (`set_time()`), and flags such as `PARALLEL_MODE`.

### Outputs You Should Expect
- Console logs showing build progress, throughput in Gbps, accommodated flow %, and stretch/coverage stats.
- Exported JSON/CSV artifacts (e.g., `Starlink_routes.json`, `Starlink_gsls.json`, `path_selection.json`, `stretch.csv`) when scripts call the corresponding `export_*` helpers.
- If using a simulator, an aggregate CSV such as `LEOConstellationSimulator.csv` capturing shell parameters plus metric summaries per job.

### Step-by-Step Recap
1. Activate the UV venv and ensure `PYTHONPATH` points to the repo root.
2. Edit or compose a script that wires together constellation shells, ground stations, traffic matrix, and optional FSPL/loss models.
3. Call `build()`, `create_network_graph()`, and `generate_routes()` on the constellation.
4. Instantiate `Throughput`, `Coverage`, `Stretch`, and invoke `build()/compute()` on each (or let `Simulator` handle it).
5. Review console output, then inspect exported JSON/CSV files for deeper analysis or visualization.
6. Repeat with different inputs to sweep the design space; scripts can be launched with `uv run python your_script.py` each time.

## 6. Batch Experiments Using Simulator
To sweep multiple constellations, write a driver similar to:
```python
from LEOCraft.simulator.LEO_constellation_simulator import LEOConstellationSimulator
sim = LEOConstellationSimulator(traffic_metrics=InternetTrafficAcrossCities.ONLY_POP_100)
# Add prebuilt constellation objects here...
sim.simulate_in_parallel()
```
Run it with `uv run python path/to/script.py`. Results land in `LEOConstellationSimulator.csv` by default.

## 7. Keeping the Environment Updated
- To add a package: `uv pip install <package>` (writes to `.venv`).
- To record the current lock: `uv pip freeze > tools/requirements-uv.txt` after testing, then commit the update.
- To reset: `rm -rf .venv` and repeat steps 2–3.

## 8. Troubleshooting Tips
- **Gurobi not found**: verify `gurobipy` is installed (step 3) and the license file is readable. Use `uv run python -c "import gurobipy; print(gurobipy.gurobi.version())"` for a quick check.
- **Missing native libs**: install system packages such as `libgl1`, `libgomp1`, or `llvm-openmp` depending on OS; UV only handles Python wheels.
- **Slow routing**: set `PARALLEL_MODE=False` if you hit fork limitations on macOS; UV environments behave like any venv in that respect.

Following these steps, you can manage LEOCraft entirely through UV while preserving the same pipeline described in `docs/link_params_and_pipeline.md`.
