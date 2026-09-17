# skylab

**Repository:** https://github.com/jcsda-internal/skylab
**Branch tracked:** develop
**Role in JEDI:** the experiment catalog — YAML definitions of experiments, algorithms, observations, models, and evaluation that EWOK consumes.

## What it is

The Skylab repo (README: "JEDI-Skylab Repository Version 8.0.0") is the
curated set of experiment definitions and supporting configuration that the
reference end-to-end JEDI experiments are built from. It is not an
executable and contains no Python package: it is the YAML library that EWOK
reads when the user runs `create_experiment.py`. From the README: *"contains
experiment definitions and additional utilities to support JEDI-Skylab
experiments."*

## How it fits into the bundle

- **EWOK** consumes Skylab's `experiments/` YAMLs, which `!INCLUDE` snippets
  from `models/`, `obs/`, `algorithms/`, and `eval/`, to generate ecFlow or
  Cylc workflows.
- **R2D2** is the data backend Skylab experiments rely on for backgrounds,
  observations, bias correction, and analyses.
- **simobs** provides the plotting/diagnostics functions used by the
  evaluation and plotting tasks Skylab experiments enable.

Skylab YAMLs locate everything through environment variables: paths are
written as `!ENV ${JEDI_WORKFLOW}/skylab/...`, `!ENV ${EWOK_WORKDIR}`,
`!ENV ${EWOK_FLOWDIR}`, and `!ENV ${EWOK_STATIC_DATA}` (static B and fix
files). `$JEDI_WORKFLOW` is the directory containing the workflow repos as
siblings — i.e. `jedi-workflow/` in this oracle layout.

## Key directories

- `experiments/` — ~80 top-level experiment YAMLs, one per scenario. Naming
  is `<model>-<algorithm>-<resolution>.yaml`: e.g. `gfs-3dvar-c24.yaml`,
  `gfs-hofx3d-c768.yaml`, `geos-3dvar-c90.yaml`, `geos-cf-4dvar-c90.yaml`,
  `mpas-3denvar-480km.yaml`, `mom6-3dvar-025deg.yaml`,
  `coupled-gfs-mom6-hofx.yaml`, `era5-analysis.yaml`, plus toy-model runs
  (`l95-*.yaml`, `qg-*.yaml`), composite "skylab" scenarios
  (`skylab-atm-land.yaml`, `skylab-aero-weather.yaml`, `skylab-marine.yaml`,
  `skylab-trace-gas.yaml`, …), `ingest-observations.yaml`,
  `*-background.yaml` (background ingest), `jedi-ctest.yaml`, and
  `workflow-engine-test.yaml`. This is what the user passes to
  `create_experiment.py`. `experiments/README.md` documents optional keys
  (`build_jedi`, `continuous`, `cycle_delay`, `preprocess obs`).
- `algorithms/` — DA algorithm building blocks: `var.yaml`, `hofx3d.yaml`,
  `hofx4d.yaml`, `forecast.yaml`, minimizers (`drpcg.yaml`, `dripcg.yaml`,
  `drplanczos.yaml`, `rpcg.yaml`, …), ensemble/static B
  (`ensembleb.yaml`, `staticb.yaml`, `saberb.yaml`, `hybridb.yaml`,
  `ensemblesolver.yaml`), HTLM (`htlm*.yaml`), `convertstate.yaml`,
  lat-lon output (`latlon*.yaml`, `writeForecastLatLon*.yaml`),
  `preprocess_obs.yaml`, `split_obsinput.yaml`, and a `BUMP/` subdir.
- `obs/` — observation configuration:
  - `obs/ufo/` — ~117 per-observation UFO YAMLs (operator + QC filters).
  - `obs/ingest/` — ~40 per-source ingest YAMLs named `*_ingest.yaml`
    (GDAS BUFR, GOES ABI G16–G19, Himawari, GNSS-RO, AIRNOW, METAR, …).
  - `obs/postProcessing/` — `ingest_monitor.yaml`, `skylab_monitor.yaml`.
  - `obs/osdf.yaml` — OSDF reader/writer configuration.
- `models/` — model-specific config trees: `gfs/`, `geos/`, `geos_cf/`,
  `mpas/`, `mom6/`, `era5/`, `ifs/`, `coupled/`, `l95/`, `qg/`. Each model
  dir holds (at least for the major models) `defaults/` (geometry,
  variables, B-matrix snippets pulled in via `!INCLUDE`), `ingest/`,
  `namelists/`, `templates/`, and `tasks/<model>.py` (EWOK model-task
  definitions referenced by the experiment's `modeltasks:` key).
- `eval/` — evaluation config: `eval_basic_lowres.yaml`,
  `eval_weatherbench_lowres.yaml`, `wb_container.def` (WeatherBench
  container), `images/`.

## Key entry points

- `experiments/<name>.yaml` — the file you pass to `create_experiment.py`.
  Anatomy: `workflow engine:` (`ecworkflow` or `cylcworkflow`), `workdir:` /
  `flowdir:` (from `$EWOK_WORKDIR`/`$EWOK_FLOWDIR`), `suite:` (path to an
  EWOK suite module, e.g.
  `${JEDI_WORKFLOW}/ewok/src/ewok/suites/cyclingDA.py`), `model:` /
  `model_path:` / `modeltasks:`, cycle controls (`init_cycle`,
  `last_cycle`, `step_cycle`, `window_length`, `window_offset`), and
  `!INCLUDE`d blocks (GEOMETRY, MODEL, SABER blocks, OBSERVATIONS, …).
- `experiments/README.md` — optional experiment keys explained.
- `algorithms/var.yaml`, `algorithms/hofx3d.yaml`/`hofx4d.yaml`,
  `algorithms/forecast.yaml` — common DA algorithms.
- `obs/ingest/*_ingest.yaml` — per-source ingest definitions;
  `obs/ufo/*.yaml` — per-instrument assimilation configs.

## Common tasks

- **Run an existing experiment** — `create_experiment.py
  $JEDI_WORKFLOW/skylab/experiments/<name>.yaml` (see
  `jedi-knowledge/ewok.md`).
- **Add a new experiment** — copy an existing `experiments/*.yaml`,
  modify fields (`workflow engine`, dates, model, observations), and PR it.
- **Pick observations to ingest** — edit
  `experiments/ingest-observations.yaml` or compose a new ingest
  experiment from `obs/ingest/*_ingest.yaml` snippets.
- **Switch from ecFlow to Cylc** — set `workflow engine: cylcworkflow`
  in your experiment YAML (default in almost all shipped experiments is
  `ecworkflow`).
- **Run continuously / near-real-time** — set `continuous: True` and
  optionally `cycle_delay:` (ISO 8601, e.g. `P1D`) or `daily_trigger:`;
  see `experiments/README.md` for engine caveats.
- **Pre-thin observations** — add `preprocess obs: True` plus a
  `_preprocess filters` section in the obs YAML (runs `ufo_preprocess.x`).

## Gotchas

- **ERA5 can now be ingested via CDS API / MARS (2026-09, skylab#972):**
  `models/mpas/tasks/runFetchBackgroundMPAS-cdsapi.py` (625 lines) adds a
  second ERA5 route for MPAS alongside the existing OSDF-hosted pull from
  `data-osdf.rda.ucar.edu` in `fetchBackgroundMPAS.py`. It checks for
  static files first and uses the F320 grid to match NCAR. Note this is
  the Open Science Data Federation sense of "OSDF" — unrelated to the
  IODA data-frame container configured by `osdf_io_pool`.
- **New experiment and eval configs (2026-09):**
  `experiments/skylab-mpas-ref.yaml`, plus
  `eval/eval_weatherbench_gfs.yaml`, `eval/eval_weatherbench_mpas.yaml`
  and `eval/observation_space_default.yaml` from the weatherbench score
  rework (skylab#954, paired with ewok#1298).
- Skylab YAMLs resolve `!ENV` references against `$JEDI_WORKFLOW`,
  `$EWOK_WORKDIR`, `$EWOK_FLOWDIR`, and (for static B / fix files)
  `$EWOK_STATIC_DATA`; EWOK additionally needs `$JEDI_SRC` and
  `$JEDI_BUILD`. All must be exported before `create_experiment.py` will
  succeed.
- Some observation ingest paths require external credentials (NASA
  Earthdata, JAXA, NCAR RDA via Globus, EUMETSAT eumdac). See `ewok.md`
  and the EWOK README for setup.
- The workflow-engine value is `ecworkflow` / `cylcworkflow` (not
  "ecflow"/"cylc") — a typo here fails experiment creation.
- **`adpsfc_ncep_prepbufr` now assimilated (2026-05):**
  `obs/ufo/adpsfc_ncep_prepbufr.yaml` was updated with expanded
  variable/channel settings so surface BUFR observations can be
  assimilated (skylab#901). If your experiments skip this type, no
  action needed; if you were working around the old incomplete YAML,
  remove the workaround.
- **Surface model updated in *_ldm.yaml obs files (2026-06, skylab#914):**
  `obs/ufo/buoy_ldm.yaml`, `metar_ldm.yaml`, `scatwind.yaml`,
  `ship_ldm.yaml`, and `synop_ldm.yaml` were all updated with a new
  surface model specification. If your experiments include any of these
  LDM observation types and you override these YAMLs locally, rebase
  against the new surface model settings.
- **Monitoring YAMLs gained vertical-level keys (2026-06, skylab#921):**
  `obs/postProcessing/ingest_monitor.yaml` and `skylab_monitor.yaml` now
  specify explicit `vertical_levels` (or `vertical_levels_range`),
  `vertical_scale`, and `vertical_log` per category (sondes, aircraft,
  surface, radar, GNSS-RO, satwind, ocean profiles), and the radiance
  section added a `seviri` member. If you maintain custom monitor YAMLs,
  mirror these keys — simobs's monitoring code consumes them.

## Further reading

- https://jedi-docs.jcsda.org/ → Inside JEDI → Skylab
- `experiments/README.md` and `obs/ingest/README.md` in the repo.
- `jedi-knowledge/ewok.md`, `jedi-knowledge/r2d2.md`,
  `jedi-knowledge/simobs.md`, `jedi-knowledge/workflow.md`
