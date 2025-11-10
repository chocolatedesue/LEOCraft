# Conda & Micromamba Workflow Guide

This note mirrors `docs/uv_pipeline_guide.md`, but focuses on running LEOCraft with Conda-class environments (either the classic Conda CLI or Micromamba). Use it whenever you want the curated environment defined in `tools/environment.yml`.

## 1. Prerequisites
- Python toolchain capable of running 3.12 (the repo’s target version).
- Conda (Miniconda or Anaconda) **or** [Micromamba](https://mamba.readthedocs.io/en/latest/user_guide/micromamba.html) installed on your machine.
- A valid `gurobi.lic` in your home directory ($HOME on Linux/macOS, `%USERPROFILE%` on Windows) so throughput LPs can solve.
- Git, curl/wget, and common build tools for your OS (OpenMP libs, libGL, etc.) in case you need to compile optional wheels.

## 2. Create the Environment
Both Conda and Micromamba can read the same `tools/environment.yml`. Pick the column that matches your tool:

| Task | Conda command | Micromamba command |
| --- | --- | --- |
| Create environment | `conda env create -f tools/environment.yml` | `micromamba create -f tools/environment.yml -n leocraft` |
| Update to latest deps | `conda env update -f tools/environment.yml --prune` | `micromamba update -f tools/environment.yml -n leocraft --prune` |
| Remove environment | `conda env remove -n leocraft` | `micromamba env remove -n leocraft` |

> Tip: if you want the env inside the repo, pass `-p .conda` (Conda) or `-p .mamba` (Micromamba) instead of `-n leocraft`.

## 3. Activate the Environment

```bash
# Conda
conda activate leocraft

# Micromamba
micromamba activate leocraft
```

Windows PowerShell users can run `conda activate leocraft` or `micromamba activate leocraft` after initializing their shell (e.g., `conda init powershell`). Once active, confirm with `python --version` → `3.12.x`.

## 4. Configure Project-Level Variables

```bash
export PYTHONPATH=$(pwd)
export GRB_LICENSE_FILE=${HOME}/gurobi.lic  # optional if license lives elsewhere
```

Set these every time you open a new shell (or add them to your shell profile). For parallel batch runs, you can also control thread usage via `export OMP_NUM_THREADS=4`.

## 5. Install / Sync Extra Python Packages

If you need an extra package temporarily:

```bash
conda install <pkg>          # or: micromamba install <pkg>
pip install <pkg>            # safe because tools/environment.yml pins Python deps
```

After validating, capture the change so teammates stay in sync:

```bash
pip freeze > tools/requirements-uv.txt  # optional mirror for UV
conda env export --from-history > tools/environment.yml
```

Only export what you truly need; avoid leaking local prefixes.

## 6. Run the Standard Pipeline

With the env active and `PYTHONPATH` set:

```bash
python examples/example_starlink.py
python -m unittest discover tests -v
coverage run -m unittest discover tests -v
```

These commands match the quick-start block in `AGENTS.md` and the README. Use the same approach for any script under `examples/` or `experiments/`, e.g.,

```bash
python examples/example_visuals3D.py
```

## 7. Batch Experiments / Simulator

The simulator APIs do not change between UV and Conda workflows. Activate the env, then run:

```bash
python path/to/your_simulation_driver.py
```

Outputs (CSV/JSON/HTML) will land wherever your script specifies (often under `./LEOConstellationSimulator.csv` or `docs/visuals/`).

## 8. Keeping Things Clean
- **List envs**: `conda env list` / `micromamba env list`
- **Check package**: `conda list gurobipy` / `micromamba list gurobipy`
- **Recreate from scratch**: remove the env, delete cached `.conda`/`.mamba` folders inside the repo if you use project-local installs, then rerun step 2.

## 9. Troubleshooting
- **Gurobi import errors**: ensure `gurobipy` shows up in `conda list` and that `GRB_LICENSE_FILE` points to a readable license. Quick test: `python -c "import gurobipy; print(gurobipy.gurobi.version())"`.
- **Missing OpenMP / GL libs**: install `libgomp1`, `libgl1`, or the macOS equivalents with your system package manager; Conda supplies most user-space deps but not kernel-level drivers.
- **Env corruption**: if `conda env update` fails mid-way, remove `.conda`/`.mamba` directories in the repo (if any) and rebuild. Persistent solver crashes often stem from stale caches under `~/.conda/pkgs`; clearing them can help.

Following these steps keeps the Conda and Micromamba flows on par with the UV workflow documented in `docs/uv_pipeline_guide.md`. Use whichever tool best matches your local deployment or CI constraints.
