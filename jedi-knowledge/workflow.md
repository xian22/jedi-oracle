# JEDI workflow stack: skylab, ewok, simobs, r2d2

How the four workflow repos in `jedi-workflow/` work together to drive
end-to-end JEDI experiments.

## At a glance

| Repo | Role in one sentence |
|------|---------------------|
| **skylab** | The catalog of experiment YAMLs and model/obs/algorithm/eval snippets. |
| **ewok** | The orchestrator — turns a Skylab YAML into an ecFlow or Cylc workflow. |
| **simobs** | The plotting/diagnostics package (maps, profiles, scorecards, monitoring) used by EWOK's evaluation tasks. |
| **r2d2** | The shared data archive (MySQL + cloud/disk stores) for inputs and outputs. |

## How they connect

A typical end-to-end experiment flows like this:

1. The user picks (or writes) a YAML in `skylab/experiments/`. The YAML
   selects an EWOK suite (`suite:` →
   `ewok/src/ewok/suites/<suite>.py`, e.g. `cyclingDA.py`), a model
   config tree (`model_path:`/`modeltasks:` → `skylab/models/<model>/`),
   and `!INCLUDE`s building blocks from `skylab/models/<model>/defaults/`,
   `skylab/obs/ufo/`, `skylab/algorithms/`, and `skylab/eval/`.
2. The user runs `create_experiment.py <yaml>` (installed from
   `ewok/src/ewok/bin/`). EWOK parses the YAML (via the external
   `yamltools` package), registers the experiment with **r2d2**
   (producing the `expid`; default lifetime `debug` = 14 days), and emits
   a workflow suite — an ecFlow `.def` plus task scripts under
   `$EWOK_FLOWDIR/<experiment_name>/`, or a Cylc `flow.cylc` (engine
   selected by `workflow engine: ecworkflow` / `cylcworkflow` in the
   YAML). `experiment_name` is `<expid>` or `<exp_name>_<expid>`.
3. The workflow engine (ecFlow or Cylc) runs the tasks. Each task wraps a
   script from `ewok/src/runtime/` (fetch/convert/run/plot scripts) using
   the scheduler headers in `ewok/src/ewok/hosts/`.
4. Tasks fetch inputs from **r2d2** (`R2D2Data.fetch(...)`), invoke
   JEDI executables built from `jedi-bundle/` (in `$JEDI_BUILD`), and
   register outputs back via `R2D2Data.store(...)`.
5. Plotting/evaluation tasks (`plotObsRun.py`, `plotAnalysisRun.py`,
   `evaluateWBRun.py`, …) import **simobs**'s `simobs.plotting`
   functions to produce maps, statistics, scorecards, and monitoring
   output from the IODA-format results.

The five required env vars (set by the user before
`create_experiment.py`; listed in the EWOK README):

- `JEDI_WORKFLOW` → the dir containing skylab/, ewok/, simobs/, r2d2/.
- `JEDI_SRC` → the dir containing jedi-bundle/'s sub-repos.
- `JEDI_BUILD` → the dir holding built JEDI executables.
- `EWOK_WORKDIR` → runtime/data dir (large; often cluster scratch).
- `EWOK_FLOWDIR` → ecFlow/Cylc suite + task script dir.

Skylab experiments also reference `EWOK_STATIC_DATA` (static B / fix
files), and EWOK sets `EWOK_CYCLE` internally at runtime. The R2D2 client
gets its database connection from `MYSQL_*` env vars.

## A typical experiment

Walking through a cycling 3DVar experiment as an example:

1. **Pick the YAML** — `$JEDI_WORKFLOW/skylab/experiments/gfs-3dvar-c24.yaml`
   sets `suite:` to `ewok/src/ewok/suites/cyclingDA.py`, points
   `model_path:` at `skylab/models/gfs/`, and `!INCLUDE`s geometry,
   variables, and SABER B-matrix blocks from
   `skylab/models/gfs/defaults/` plus observation configs from
   `skylab/obs/ufo/`.
2. **Create the suite** —
   `create_experiment.py $JEDI_WORKFLOW/skylab/experiments/gfs-3dvar-c24.yaml`.
   EWOK registers the experiment in R2D2 and writes the suite to
   `$EWOK_FLOWDIR/<experiment_name>/`.
3. **The workflow runs** — tasks fetch backgrounds and observations via
   R2D2 (ingest experiments populate it from external sources using
   `skylab/obs/ingest/*_ingest.yaml` and EWOK's
   `fetchObservationsRun-*`/`convertObservationsRun-*` scripts), run the
   variational solver from `oops` + model-specific code (`fv3-jedi`,
   `mpas-jedi`, `soca`), and produce analyses/feedback files.
4. **Outputs land in `$EWOK_WORKDIR/<experiment_name>/`** and are
   registered in R2D2 for later use.
5. **Diagnostics + visualization** — `skylab/eval/` building blocks drive
   evaluation tasks (incl. WeatherBench scoring); plotting tasks call
   `simobs.plotting` functions; `skylab_monitor` provides experiment
   monitoring.

## When to consult which brief

- **`skylab.md`** — when you're choosing or modifying an experiment YAML,
  picking algorithms, or selecting observations.
- **`ewok.md`** — when you're running `create_experiment.py`, debugging
  workflow generation, dealing with ecFlow/Cylc, or setting up env vars
  and ingest authentication.
- **`simobs.md`** — when you're dealing with plots, scorecards,
  monitoring, or the synthetic GOES ABI file generator.
- **`r2d2.md`** — when you're fetching, storing, searching, or deleting
  data, or troubleshooting "where is my file?" questions.
- **This file** — when you want the big picture or aren't sure which of
  the four to start with.

## Relationship to the bundle

The workflow stack invokes JEDI executables built from the bundle. In
particular:

- Variational and ensemble DA — `oops` + per-model libs
  (`fv3-jedi`, `mpas-jedi`, `soca`).
- Forward operators — `ufo` (with `crtm` and/or `rttov` as backends),
  run inside hofx/DA tasks.
- Observation I/O — `ioda`; observations are converted into IODA
  format by converters (`iodaconv`, `bufr2ioda`, `obs2ioda`, …) driven
  by EWOK's `convertObservationsRun-*` scripts.
- Linear models for 4DVar — `fv3-jedi-lm` for FV3, the bundle's MPAS
  components for MPAS, etc.
- Background-error covariances — `saber`, optionally with `gsibec`.

So a "skylab experiment" is really skylab+ewok+r2d2 invoking the
bundle's executables in a defined order with carefully managed inputs
and outputs, with simobs rendering the results.

## Recent cross-repo changes

- **WeatherBench scoring reworked (2026-09, skylab#954 + ewok#1298).**
  A paired change: ewok's `src/runtime/saveScoresWBRun.py` and test defs
  on one side, and on the other skylab's new
  `eval/eval_weatherbench_gfs.yaml`, `eval/eval_weatherbench_mpas.yaml`
  and `eval/observation_space_default.yaml`. Because it spans both repos,
  update skylab and ewok together — a mismatched pair will fail at the
  scoring task.
- **ERA5 MARS/CDS ingest for MPAS (2026-09, skylab#972).** A second ERA5
  fetch route alongside the OSDF-hosted one. See `jedi-knowledge/skylab.md`.
- **simobs plotting and naming updates (2026-09).** See `jedi-knowledge/simobs.md`.

## Further reading

- https://jedi-docs.jcsda.org/ → Inside JEDI → Skylab.
- `jedi-knowledge/skylab.md`, `jedi-knowledge/ewok.md`,
  `jedi-knowledge/simobs.md`, `jedi-knowledge/r2d2.md`.
- `jedi-knowledge/jedi-bundle.md` for the build side.
