# oops

**Repository:** https://github.com/jcsda-internal/oops
**Branch tracked by bundle:** develop
**Role in JEDI:** Foundational C++/Fortran framework providing the abstract data assimilation algorithms, interfaces, and toy models that every model-specific JEDI repo (fv3-jedi, mpas-jedi, soca, ufo, ioda, saber, vader, ...) plugs into.

## What it is

OOPS (Object Oriented Prediction System) is the core of JEDI. It defines the
templated C++ class hierarchy for states, increments, geometries, models,
observation operators, error covariances, minimizers, and full DA
applications (3DVar, 4DVar, EnVar, EnKF/LETKF/GETKF, hybrid). Concrete
models (e.g. fv3-jedi, mpas-jedi, soca) implement a `Traits` struct that
satisfies the OOPS interfaces; OOPS then composes them into runnable
applications. OOPS itself is model-agnostic.

OOPS originates from ECMWF's OOPS prototype and is co-developed with JCSDA.
It also ships two reference / toy models (Lorenz-95 and the QG
shallow-water model) used as the canonical regression and demonstration
platform for new algorithms before they are wired into operational models.

## How it fits into the bundle

- **Depended on by:** essentially everything else in the bundle — ufo,
  ioda, saber, vader, fv3-jedi, mpas-jedi, soca, coupling. They
  `find_package(oops 1.10.0 REQUIRED)`.
- **Depends on:** eckit (≥1.24.4, MPI component), fckit (≥0.11.0),
  atlas (≥0.35.0; `hic` is also required for atlas ≥0.39), MPI, NetCDF
  (parallel build required), Eigen3, Boost, LAPACK/MKL, optionally GPTL
  (profiling) and nlohmann_json (+schema validator, for JSON-schema
  output).
- **Build position:** built very early in the bundle (after
  eckit/fckit/atlas/jedicmake). See top-level `jedi-bundle/CMakeLists.txt`.

## Key directories

- `src/oops/` — the OOPS library proper (the `oops::` namespace).
  - `base/` — core abstract classes: `State`, `Increment`, `Model`,
    `Geometry`, `Variables`/`Variable`, `ObsVariables`, `ObsSpaceBase`,
    `Locations`, `LinearModelBase`, plus `FieldSet3D`/`FieldSet4D`/
    `FieldSets`, ensemble inflation (`RTPP.h`, `RTPS.h`,
    `InflationBase.h`), covariance bases
    (`ModelSpaceCovarianceBase.h`, `EnsembleCovariance.h`,
    `HybridCovariance.h`) and `instantiateCovarFactory.h`.
  - `interface/` — templated interface wrappers (`Geometry<MODEL>`,
    `State<MODEL>`, `LinearObsOperator<OBS>`, ...) that adapt
    model-specific implementations to the generic algorithms.
  - `assimilation/` — cost functions (`CostFct3DVar`, `CostFct4DVar`,
    `CostFct4DEnsVar`, `CostFctFGAT`, `CostFctWeak`), minimizers (DRPCG,
    PCG, GMRES family, MINRES, Lanczos, saddle-point, LETKF/GETKF
    solvers), and as of 2026-05 the sequential ensemble solver framework
    (`SequentialEnsembleSolver.h`, `EAKFSolver.h`).
  - `runs/` — ~30 top-level `Application` classes: `Variational`,
    `HofX3D`, `HofX4D`, `Forecast`, `AdjointForecast`,
    `EnsembleApplication`, `TemplatedEnsembleApplication`,
    `LocalEnsembleDA`, `EnsembleGETKFApplication`, `ControlPert`,
    `ConvertState`, `ConvertIncrement`, `ConvertToStructuredGrid`,
    `EnsMeanAndVariance`, `EnsRecenter`, `EnsembleInflation`,
    `RescaleEnsPerts`, `GenEnsPertB`, `GenHybridLinearModelCoeffs`,
    `HybridGain`, `TLMToolbox`, `SqrtOfVertLoc`,
    `InterpolateStateBetweenModels`, `AddIncrement`, `DiffStates`,
    `ExternalDFI`, `Test`. These are what model executables instantiate.
  - `generic/` — model-agnostic implementations: `PseudoModel`,
    `IdentityModel`, `HybridLinearModel` + HTLM machinery
    (`HtlmCalculator`, `HtlmEnsemble`, `HtlmRegularization`,
    `HybridLinearModelCoeffs`, `SimpleLinearModel*`), `Diffusion`,
    `VerticalLocEV`, `UnstructuredInterpolator`, `AtlasInterpolator`,
    `GlobalInterpolator`, `gc99`/`soar` correlation functions, FFT
    helpers, `instantiateModelFactory.h` /
    `instantiateLinearModelFactory.h`.
  - `coupled/` — generic coupled-model infrastructure
    (`GeometryCoupled`, `ModelCoupled`, `StateCoupled`, `TraitCoupled`,
    ...).
  - `util/` — `DateTime`, `Duration`, `Logger`, `Printable`, `Factory`,
    `Parameters`/JSON-schema infrastructure, `AtlasArrayUtil`, FieldSet
    helpers, `Random`, `abor1_ftn`.
  - `mpi/` — MPI scope/communicator helpers.
  - `atlas/` — thin Atlas interpolator wrapper.
  - `contrib/` — DCMIP test initial conditions.
- `src/test/` — the generic test framework (`TestEnvironment.h`,
  `TestFixture.h`) and the per-interface test suites in
  `src/test/interface/` (`Geometry.h`, `State.h`, `Increment.h`,
  `Model.h`, `ObsSpace.h`, ...) that downstream model repos instantiate
  with their `Traits` to validate an implementation. Sub-dirs mirror
  `src/oops/` (`assimilation/`, `base/`, `coupled/`, `generic/`,
  `mpi/`, `util/`, plus `testinput/`).
- `l95/` — Lorenz-95 toy model (`l95/src/lorenz95/`,
  `l95/src/executables/`, `l95/test/`).
- `qg/` — Quasi-Geostrophic shallow-water toy model. `qg/model/`,
  `qg/mains/`, `qg/test/`.
- `coupled/` — tests for the L95+QG coupled toy at the bundle level.
- `share/oops/` — installed shared assets (e.g. `suppressions/` for
  valgrind/sanitizers).
- `tools/` — `cpplint.py`, `compare.py` (test ref comparison),
  `gen_yaml_ensemble.sh`, `plot.py`, `test_wrapper.sh`.
- `docs/` — Doxygen configuration; built only with `-DENABLE_OOPS_DOC=ON`.

## Key entry points

- `src/oops/runs/Run.cc` / `Run.h` — the harness every JEDI executable
  starts with (`oops::Run run(argc, argv); return run.execute(application);`).
- `src/oops/runs/Variational.h` — variational DA driver (3DVar, 4DVar,
  EnVar, hybrid).
- `src/oops/assimilation/SequentialEnsembleSolver.h` — new base class
  for sequential ensemble solvers (LETKF/GETKF/EAKF share this).
- `src/oops/assimilation/EAKFSolver.h` — new EAKF implementation (87 lines).
- `src/oops/runs/HofX4D.h`, `HofX3D.h` — observation forward operator
  applications.
- `src/oops/runs/LocalEnsembleDA.h` — EnKF / LETKF / GETKF entry point.
- `src/oops/assimilation/CostFct4DVar.h`, `CostFct3DVar.h`,
  `CostFct4DEnsVar.h` — cost-function definitions.
- `src/oops/base/Variables.h`, `base/Variable.h` — variable-naming
  abstraction every model must speak.
- `src/oops/util/parameters/` — `Parameters` / JSON-schema-aware YAML
  configuration system; understanding this unlocks every JEDI yaml.
- `qg/mains/qg4DVar.cc` — canonical example of how a model wires its
  `Traits` into `Variational<QG>`.

## Common tasks

- **Run the QG 4DVar reference test** — configure with
  `-DENABLE_QG_MODEL=ON` (default), then `ctest -R qg_4dvar`.
- **Run the L95 reference tests** — `-DENABLE_LORENZ95_MODEL=ON`
  (default), `ctest -R l95_`.
- **Add a new DA application** — new header in `src/oops/runs/` deriving
  from `oops::Application`, register in `src/oops/runs/CMakeLists.txt`,
  then instantiate from the model's `mains/` (template:
  `qg/mains/qg4DVar.cc`).
- **Add a new minimizer** — drop a header in `src/oops/assimilation/`
  following `DRPCGMinimizer.h` / `PCGMinimizer.h`, register it in
  `Minimizer.h`'s factory.
- **Wire YAML <-> code** — `oops::Parameters`
  (`src/oops/util/parameters/`) still exists, but new oops-core code
  increasingly takes `eckit::Configuration` directly (see the Parameters
  phase-out gotcha below). JSON-schema is auto-emitted via
  `cmake/oops_output_json_schema.cmake`.
- **Validate a new model interface** — instantiate the generic
  interface tests from `src/test/interface/` with your model's
  `Traits` (every model repo's `test/` executables follow this
  pattern).
- **Time/date arithmetic** — `oops::DateTime` and `oops::Duration`
  (`src/oops/util/DateTime.h`, `Duration.h`).
- **Logging** — `oops::Log::info()/debug()/test()/trace()` from
  `src/oops/util/Logger.h`.

## Gotchas

- **Interpolator caches moved out of `GeometryData` (2026-09, oops#3333):**
  `src/oops/base/GeometryData` shed ~539 lines. The interpolator-related
  caching now lives in new generic components:
  `src/oops/generic/MeshTriangulation.{h,cc}`,
  `ProximitySearch.{h,cc}` and `SourceProximityPartitioner.{h,cc}`.
  Model interfaces that reached into `GeometryData` for interpolation
  internals must be updated — `jedi-knowledge/saber.md` (saber#1303) and
  `jedi-knowledge/pyiri-jedi.md` (pyiri-jedi#185) already were.
- **Common Parameter types explicitly instantiated (2026-09, oops#3382):**
  `src/oops/util/parameters/ParameterInstantiations.cc` adds explicit
  instantiations of the common `Parameter`/`ParameterTraits` types to cut
  compile time. If you add a new Parameter type and hit a link error,
  check whether it needs an entry there.
- OOPS requires `NETCDF4_PARALLEL` — a serial-only NetCDF build fails
  configure with `Missing PARALLEL feature for NetCDF`.
- Heavy template use; compile errors can be huge. Build the toy models
  first to validate the stack before iterating on a model interface.
- `oops::Variables` / `Variable` API is evolving (stricter typing in
  1.10.x); old downstream code using `Variables(std::vector<std::string>)`
  may need updates.
- Downstream pins `find_package(oops 1.10.0 REQUIRED)`; bumping
  `project(... VERSION ...)` here means bumping pins everywhere.
- The top-level `coupled/` subdirectory builds only when both QG and L95
  are enabled.
- **QC flag API change (2026-05):** `ObsDataVector` QC flags are now
  passed as references rather than shared pointers across the obs filter
  stack. Any downstream code holding a `shared_ptr` to the QC flag vector
  from `ObsFilter` / `ObsFilters` will need updating (see ufo#4112 /
  oops#3267).
- **`ModelData::defaultVariables` is no longer static (2026-05):** call
  it on a model instance, not the class (oops#3244).
- **`UnstructuredInterpolator` now fills outside regional domains
  (2026-05):** behaviour changed — values outside the domain are
  extrapolated rather than left undefined (oops#3264).
- **RTPS is now incremental (2026-05):** `src/oops/base/RTPS.h` was
  refactored; if you have local branches touching ensemble inflation,
  check for conflicts (oops#3278).
- **Implicit vertical diffusion added to `Diffusion` (2026-05):**
  `src/oops/generic/Diffusion.{h,cc}` gained an implicit-scheme
  option (oops#3275). The corresponding SABER block and UFO error model
  were extended in lockstep — test YAMLs exist in saber and jedi-docs.
- **`ObsVariables::dimList()` added (2026-05):** new member function
  on `src/oops/base/ObsVariables.h` (oops#3281); downstream `put_db`
  callers in ufo and ioda were also updated — local branches touching
  `ObsSpace::put_db` calls must add the `dimList` argument.
- **QC now applied to ensemble mean and modulated members in GETKF
  (2026-05):** `GETKFSolver.h` and `LocalEnsembleSolver.h` updated
  (oops#3231). Check test references if running l/getkf experiments.
- **Parameters phase-out (2026-05):** oops#3296 removed the last
  `Parameters` usage from the oops test suite; core oops code is
  migrating from `oops::Parameters` subclasses toward plain
  `eckit::Configuration`. The `util/parameters/` infrastructure is
  still installed (downstream repos use it heavily), but don't model
  new oops-core code on `Parameters`.
- **GlobalInterpolator sparse comms (2026-06):**
  `src/oops/generic/GlobalInterpolator.cc` now switches from
  `allToAllv` to point-to-point sends when the communication graph is
  sparse (oops#3271) — relevant when debugging interpolation MPI
  behaviour or performance.
- **`isRegional` safety check removed from `UnstructuredInterpolator`
  (2026-06):** oops#3313 dropped the regional-domain guard from
  `UnstructuredInterpolator` and the related helpers in
  `util/FunctionSpaceHelpers`; combined with oops#3264 (extrapolation
  outside regional domains) the interpolator no longer special-cases
  regional grids.

## Further reading

- In-repo: `README.md`, `docs/Doxyfile.in`.
- jedi-docs.jcsda.org: "OOPS" section under JEDI Components.
- Related briefs: `jedi-knowledge/ufo.md`, `jedi-knowledge/ioda.md`,
  `jedi-knowledge/saber.md`, `jedi-knowledge/vader.md`,
  `jedi-knowledge/fv3-jedi.md`, `jedi-knowledge/mpas-jedi.md`,
  `jedi-knowledge/soca.md`, `jedi-knowledge/coupling.md`,
  `jedi-knowledge/jedi-bundle.md`.
