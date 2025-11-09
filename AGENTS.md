# Repository Guidelines

## Project Structure & Module Organization
- `LEOCraft/` holds the production modules—`constellations/`, `satellite_topology/`, `attenuation/`, `simulator/`, `user_terminals/`, `visuals/`. Keep new logic in the closest subpackage so imports remain stable.
- `examples/` hosts runnable constellation scripts (e.g., `example_starlink.py`) that double as living tutorials and smoke tests.
- `dataset/` (CSVs, JSON traffic) and `docs/visuals/` are the only approved locations for large static artifacts.
- `tests/` mirrors the package tree; add `tests/test_<module>.py` for every feature. `experiments/` is for notebooks and parameter sweeps only.

## Build, Test, and Development Commands
```
conda env create -f tools/environment.yml   # one-time environment setup
conda activate leocraft                     # enter curated toolchain
export PYTHONPATH=$(pwd)                    # enable in-place imports
python examples/example_starlink.py         # full simulation sanity run
python -m unittest discover tests -v        # run entire test suite
coverage run -m unittest discover tests -v  # collect coverage data
coverage html && open htmlcov/index.html    # inspect coverage locally
```

## Coding Style & Naming Conventions
- Follow PEP 8 with 4-space indentation, `snake_case` modules/functions, and `CamelCase` classes (`LEOConstellation`, `GroundStation`).
- Prefer explicit imports from `LEOCraft.<package>.<module>` so tooling can resolve dependencies.
- Preserve descriptive keywords (`altitude_m`, `inclination_degree`) when extending APIs, and emit log lines in the existing `[Component] Message...` style.

## Testing Guidelines
- Keep using the built-in `unittest` framework housed in `tests/`.
- Name cases `Test<Component>` and methods `test_<behavior>`; align filenames with the source module path.
- While iterating, run `python -m unittest -v tests/<file>.py`, then rerun the full suite plus `coverage report` before opening a PR.
- Treat coverage regressions as blockers and lift touched files back to their previous percentages.

## Commit & Pull Request Guidelines
- Recent history (`Add cite this work`) uses short, imperative subjects; follow `Verb object` formatting under ~60 characters.
- Reference related issues in the body, summarize functional/performance impact, and flag new datasets or visualization outputs.
- PRs should list reproduction steps (`python examples/...`), attach relevant plots from `docs/visuals/`, and include screenshots or metrics when outputs change.

## Security & Configuration Tips
- Gurobi powers throughput LPs; obtain a named-user academic license, run `./grbgetkey <LICENSE_KEY>`, and place `gurobi.lic` in your home directory before solver-heavy runs.
- Never commit license keys, proprietary datasets, or generated `htmlcov/` assets. Extend ignore rules instead.\n*** End Patch"}{assistant to=functions.apply_patch.outputs.quoted code block code```json
