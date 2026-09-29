# ewok

**Repository:** https://github.com/jcsda-internal/ewok
**Branch tracked:** develop
**Role in JEDI:** the orchestrator — turns Skylab experiment YAMLs into runnable ecFlow or Cylc workflows.

## What it is

EWOK = **Experiments and Workflows Orchestration Kit** (Python package,
`setup.py` version 0.8.0). EWOK reads an experiment YAML (typically from
Skylab), generates a workflow suite, and submits it to a workflow engine.
The primary engine is [ecFlow](https://confluence.ecmwf.int/display/ECFLOW)
(`workflow engine: ecworkflow`); [Cylc](https://cylc.github.io/) support
(`workflow engine: cylcworkflow`) is functional but still listed as under
development in the README.

## How it fits into the bundle

- **Reads from `skylab/`** — experiment YAMLs and the `models/`, `obs/`,
  `algorithms/`, `eval/` snippets they `!INCLUDE`.
- **Reads/writes via `r2d2`** — `src/ewok/r2d2_util.py` (`R2D2Util`)
  registers each experiment in R2D2 at creation time (default lifetime
  `debug`, 14 days); runtime tasks fetch/store data through the R2D2
  client.
- **Calls JEDI executables** in `$JEDI_BUILD` (built from `jedi-bundle/`).
- **Imports `simobs`** in its plotting/diagnostics runtime scripts
  (`plotObsRun.py`, `plotAnalysisRun.py`, `plotEnsStatsRun.py`, …).
- Uses **`yamltools`** (a separate JCSDA Python package, not part of this
  repo) for YAML parsing, template substitution, and date formatting.

EWOK is the user-facing entry point; once an experiment is created, the
workflow engine takes over.

## Key directories

- `src/ewok/` — the main package:
  - `bin/create_experiment.py` — the CLI (installed as a script).
  - `bin/set_pyioda_path.py` — helper installed alongside.
  - `Experiment.py` — `Experiment` class: walks the suite definition,
    adds tasks to the engine, manages cycles/families.
  - `get_suite.py`, `get_model.py` — load the suite module (from the
    experiment's `suite:` key) and the model-task module (`modeltasks:`).
  - `r2d2_util.py` — `R2D2Util.register_experiment()`, queue parameters,
    experiment start/end date bookkeeping.
  - `suites/` — suite definitions: `cyclingDA.py`, `cyclingEDA.py`,
    `hofx.py`, `forecast.py`, `ensembleSolver.py`, `bTraining.py`,
    `convertStates.py`, `ingestObservations.py`, `ingestBackground.py`,
    `obsBench.py`, `plotobs.py`, `skylab.py`, `jediTest.py`,
    `testWorkflow.py`.
  - `tasks/` — ~70 task classes (`Task.py`, `JediTask.py`,
    `CompositeTask.py`, `NoTask.py`, plus concrete tasks:
    `fetchObservations.py`, `convertObservations.py`, `getBackground.py`,
    `hofx.py`, `forecast.py`, `ensembleSolver.py`, `plot*.py`, …).
  - `workflows/` — engine backends: `Workflow.py` base,
    `ecflow/ecflow.py` (+ `ewoktask.ecf`, `head.h`, `tail.h`),
    `cylc/cylc.py` (+ `ewoktask.sh`, `head.sh`, `tail.sh`, `cylctools/`).
  - `hosts/` — scheduler headers per platform: `slurm`, `pbs`,
    `noscheduler`, `slurm_discover`, `slurm_aws-pcluster`, each in `.h`
    (ecFlow) and `.cylc` flavors.
- `src/runtime/` — the scripts executed inside workflow tasks:
  `jedi-run.sh`/`jedi-build.sh`/`jedi-source.sh`,
  `fetchObservationsRun-{s3,s3_secure,rda,wget,cp,api,eumdac,file_search}.sh`,
  `convertObservationsRun-*.sh` (one per converter: bufr2ioda, obs2ioda,
  goes2ioda, gnssro_bufr2ioda, …), `get*Run.py`, `plot*Run.py`,
  `evaluateWBRun.py`, `endCycleRun.py`, etc.
- `test/` — pytest suite verifying workflow generation for both engines:
  - `test_cylc_flow_generation.py`, `test_ecflow_def_generation.py` —
    parse experiment YAMLs, build the full suite DAG, compare against
    baselines.
  - `test/testdata/` — paired baselines `<exp>.def` + `<exp>_flow.cylc`
    for `gfs-3dvar-c24`, `gfs-hofx-c24`, `jedi-ctest`,
    `skylab-atm-land-small`, `workflow-engine-test`.
  - `conftest.py` — shared fixtures/mocks; `test/README.md` explains how
    to run and regenerate baselines.
- `environment.yml` — conda environment for users not on a spack-stack
  platform (ecflow, jinja2, ruamel.yaml, cartopy, matplotlib, pytest, …).

There is no `experiments/` directory in ewok itself — example experiments
live in `skylab/experiments/`.

## Key entry points

- `create_experiment.py <experiment_file>.yaml` — primary CLI (installed
  from `src/ewok/bin/` by `setup.py`). Flags: `-ns/--no-submit` (generate
  only), `-sus/--suspend` (suspend init/final tasks; for tutorials),
  `--test` (create CI test suite). It parses the YAML with `yamltools`,
  registers the experiment with R2D2 (producing the `expid`), then builds
  the suite. If `exp_name:` is set in the YAML, the experiment directory
  is named `<exp_name>_<expid>`, otherwise just `<expid>`. A `resume:`
  block restarts an existing experiment from a given cycle.
- `src/ewok/suites/<suite>.py` — referenced by the experiment YAML's
  `suite:` key; defines the DAG.

## Required environment

The README lists five required env vars:

- `JEDI_WORKFLOW` — directory containing the workflow source repos
  (`ewok`, `r2d2-client`, `simobs`, `skylab`).
- `JEDI_SRC` — directory containing `jedi-bundle/`'s sub-repos.
- `JEDI_BUILD` — built JEDI executables.
- `EWOK_WORKDIR` — runtime/data dir for experiments (large, often
  cluster-mounted scratch; compute-node accessible).
- `EWOK_FLOWDIR` — ecFlow/Cylc suite definitions and task scripts
  (smaller; server/login-node accessible).

Skylab experiments additionally reference `EWOK_STATIC_DATA` (static B /
fix files). At runtime EWOK sets `EWOK_CYCLE` internally for cycle-aware
tasks. For Cylc, also configure `~/.cylc/flow/global.cylc` (symlink dirs,
platforms) per the JEDI Knowledge Base notes linked in the README.

## Key paths after creation

- ecFlow: suite `.def` + task scripts in `$EWOK_FLOWDIR/<experiment_name>/`;
  runtime data in `$EWOK_WORKDIR/<experiment_name>/`; job logs under
  `$EWOK_FLOWDIR/` (ECF_JOBOUT).
- Cylc: generated `flow.cylc` in `$EWOK_FLOWDIR/<workflow_name>/`; run dir
  `$EWOK_FLOWDIR/cylc-run/<workflow_name>/run1/` (symlinked from
  `~/cylc-run/` via global.cylc); logs in
  `~/cylc-run/<workflow_name>/run1/log/`.

## Common tasks

- **Create + submit an experiment** — `create_experiment.py
  $JEDI_WORKFLOW/skylab/experiments/<name>.yaml`.
- **Generate the suite without submitting** — add `-ns`.
- **Suspend automatic completion** (training/tutorials) — add `-sus`.
- **Generate CI test suite** — add `--test`.
- **Start the ecFlow server** — `ecflow_start.sh`; UI: `ecflow_ui &`.
- **Cylc UIs** — `cylc gui` or `cylc tui`.
- **Run the workflow-generation tests** — `pytest ewok/test/ -v`
  (or `-k <experiment>` for one experiment across both engines).

## Gotchas

- **cylc: abort on timeout, not on stall (2026-09, ewok#1320):** cylc
  experiments no longer abort when the workflow stalls; they abort on
  timeout instead. ObsBias files are now published (ewok#1319), and the
  `exclusive_nodes` option is implemented here (ewok#1322, see
  `jedi-knowledge/skylab.md`).
- The required env vars are not optional. Missing any one of them fails
  experiment creation in confusing ways.
- ecFlow server-side and client-side hostnames must match — when the UI
  doesn't see the server, add it manually under
  *Servers → Manage servers* using the host/port from `ecflow_start.sh`.
- Cylc support requires extra Python dependencies and a configured
  `global.cylc`; without `symlink dirs` everything lands in `~/cylc-run/`.
- Globus/RDA, NASA Earthdata, JAXA AMSR2, and EUMETSAT (eumdac) ingest
  each have their own auth setup. The README has detailed walkthroughs
  (sections "EWOK Ingest …") — start there rather than improvising. For
  RDA fetch failures, check
  `$EWOK_FLOWDIR/<exp_id>/ingest/obs_<obsname>/fetchObservations_<obsname>.1`.
- **Common prepbufr handling refactored (2026-05):**
  `src/ewok/tasks/convertObservations.py` and `storeObsBias.py` were
  updated to share common prepbufr handling logic (ewok#1282). If you
  have local branches touching these task files, check for merge
  conflicts.

## Further reading

- README in this repo (extensive — covers ingest auth, key paths, Cylc
  setup, troubleshooting) and `test/README.md`.
- https://jedi-docs.jcsda.org/ → Inside JEDI → Skylab.
- `jedi-knowledge/skylab.md`, `jedi-knowledge/r2d2.md`,
  `jedi-knowledge/simobs.md`, `jedi-knowledge/workflow.md`.
