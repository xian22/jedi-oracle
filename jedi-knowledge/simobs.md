# simobs

**Repository:** https://github.com/jcsda-internal/simobs
**Branch tracked:** develop
**Role in JEDI:** the plotting/diagnostics package for Skylab experiments — pure-Python tools that read IODA-format output and produce maps, profiles, scorecards, and monitoring plots, plus a synthetic-observation file generator.

## What it is

The README still introduces SIMOBS as *"a high-performance stand-alone
operation and visualization application for the JCSDA UFO and CRTM
models"*, but the README body is essentially empty and the code tells the
real story: `setup.py` (version 1.6.0) describes it simply as **"Plotting
tools for JEDI"**. In practice simobs is a pure-Python package
(numpy/h5py/matplotlib/cartopy/netCDF4/pandas/scipy — no compiled
extensions, no direct UFO/CRTM linkage) whose `simobs.plotting` functions
are imported by EWOK's plotting and evaluation runtime tasks, and whose
`simobs.generator` can write synthetic observation files.

## How it fits into the bundle

- **EWOK** imports `simobs.plotting` inside its runtime scripts —
  `ewok/src/runtime/plotObsRun.py`, `plotAnalysisRun.py`,
  `plotEnsStatsRun.py`, `plotBasicScoresRun.py`,
  `compareDiagnosticsRun.py`, `saveFeedbackRun.py` — so every Skylab
  experiment that enables plotting/monitoring tasks runs simobs code.
- **Reads JEDI output** (HofX feedback, analyses) in IODA netCDF/HDF5
  format via `src/simobs/plotting/read_ioda_netcdf.py`; it consumes what
  UFO/CRTM produced rather than wrapping those libraries.
- **R2D2** is where the plotted inputs come from and where `media`
  items (plots) are registered when EWOK drives the tasks.

## Key directories

- `src/simobs/plotting/` — the real content of the package:
  - map plots: `plot_global_scatter_map.py`, `plot_conus_scatter_map.py`,
    `plot_global_latlon_map.py`, `plot_maps.py`.
  - statistics: `plot_stats.py`, `plot_rad_stats.py`, `plot_profiles.py`,
    `plot_time_bar.py`, `plot_score_cards.py`.
  - Skylab monitoring/diagnostics: `skylab_monitor.py`,
    `skylab_convergence.py`, `skylab_desroziers.py`,
    `skylab_ensstats.py`.
  - I/O and helpers: `read_ioda_netcdf.py`, `plot_get_functions.py`,
    `plot_utils.py`, `define_radiometer.py`, `define_radiosonde.py`,
    `define_wmo_platform.py`, `fix_nchans.py`, `fix_height.py`
    (the last two are installed as console scripts).
  - `cartopy_data/` — bundled shapefiles (packaged via `package_data`).
- `src/simobs/generator/` — `generate_goes_abi_file.py`, a script that
  writes a synthetic `abi_global` GOES file.
- `src/simobs/config.py`, `src/simobs/custom_funcs.py` — currently empty
  placeholder modules.
- `data/` — placeholder `input/`, `output/`, `temporary/` dirs (contents
  gitignored).
- `notebooks/` — exists but is empty in git (contents gitignored); do not
  point users there for examples.
- `test/` — `skylab-plots.py` (exercises the plotting entry points),
  `test_pep8.py`, `tox.ini`.

## Key entry points

- `from simobs.plotting.<module> import <func>` — the public API; the
  cleanest worked examples of correct call signatures are EWOK's runtime
  scripts (e.g. `ewok/src/runtime/plotObsRun.py` imports `plot_global`,
  `plot_conus`, and `skylab_monitor.main`).
- `setup.py` — installs the `simobs` package plus the `fix_nchans.py` and
  `fix_height.py` scripts.
- `src/simobs/generator/generate_goes_abi_file.py` — synthetic GOES ABI
  file generation.
- `test/skylab-plots.py` — a runnable exercise of the plot functions.

## Common tasks

- **Generate experiment plots** — normally done by EWOK plotting tasks
  (enabled in the Skylab experiment YAML); for ad-hoc use, import the
  relevant `simobs.plotting` module and mimic the calls in
  `ewok/src/runtime/plot*Run.py`.
- **Monitor a running Skylab experiment** — `skylab_monitor.py` (used by
  `skylab/obs/postProcessing/skylab_monitor.yaml` flows).
- **Create synthetic ABI observations** —
  `src/simobs/generator/generate_goes_abi_file.py`.

## Gotchas

- **Plotting and naming updates (2026-09):** longer timeseries now get a
  legible x-axis (simobs#409); GNSSRO writes one logfile per satellite
  (#405); names with gnssro and snake_case qualifiers are accepted (#407);
  additional `insitu_2d` data types recognised (#385).
- The README is effectively empty — the canonical reference is the Python
  source in `src/simobs/plotting/` and the EWOK runtime scripts that call
  it.
- `setup.py` pins `Shapely==1.8.0`; newer Shapely in your environment can
  break cartopy interactions.
- Despite the repo's name and README blurb, there is no observation
  *simulation* of the UFO/CRTM kind here (the only generator is the
  synthetic GOES ABI file writer) — for actual H(x) computation use the
  JEDI executables via EWOK's hofx suites.
- **Plotting keyword-arg changes (2026-05, simobs#379/#373):**
  `plot_time_bar.py` x-axis corrected for initial files not starting at
  00 UTC; `plot_global_scatter_map.py`, `plot_maps.py`,
  `plot_rad_stats.py`, `skylab_monitor.py` gained a `map_center` keyword
  and other keyword-arg cleanup. If you call these directly with
  positional args, check that the call signature still matches.
- **GOES ABI global synthetic file generator (2026-06, simobs#387):**
  `src/simobs/generator/generate_goes_abi_file.py` added — use this when
  you need synthetic `abi_global` observations for testing without real
  satellite data.
- **`table_group_pass` rewritten (2026-06, simobs#389):**
  `plot_utils.table_group_pass` no longer mutates the caller's DataFrame
  (the old version did `dropna(inplace=True)` and added columns), takes
  a new `obserror=` keyword, and computes multi-experiment differences
  on aggregated per-group statistics instead of the old per-observation
  `merge_asof` matching — thinning/QC differences between experiments
  made 1:1 obs matching unreliable. Diff columns are now
  `(<group>_dif, mean/std)` on the aggregated table, with the base
  experiment carrying 0/0 (so the t-test reduces to a one-sample test).
  Callers relying on the old side effects or the per-ob diff columns
  must update; `plot_profiles.py` and `skylab_monitor.py` were adjusted
  in the same PR.

## Further reading

- `src/simobs/plotting/README.md` in the repo.
- `jedi-knowledge/ufo.md`, `jedi-knowledge/crtm.md` for the actual
  forward-operator code.
- `jedi-knowledge/ewok.md`, `jedi-knowledge/skylab.md`,
  `jedi-knowledge/workflow.md`.
