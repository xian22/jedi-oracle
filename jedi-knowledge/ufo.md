# ufo

**Repository:** https://github.com/jcsda-internal/ufo
**Branch tracked by bundle:** develop
**Role in JEDI:** Unified Forward Operator — the observation-space side of JEDI. Houses every observation operator (the forward / TL / AD operations from model state to simulated obs), every QC filter, bias-correction predictors, and the OBS Traits adapter that lets oops-side algorithms drive observations.

## What it is

UFO provides JEDI's `ObsTraits` and the bulk of observation-space code:

- **Forward operators** (`operators/`) — organized by operator *type*, not
  by instrument: generic interpolators (`vertinterp/`, `identity/`,
  `atmvertinterplay/`), radiance backends (`crtm/`, `rttov/`), GNSS-RO
  (`gnssro/`), ground-based GNSS (`groundgnss/`), radar
  (`radardopplerwind/`, `radarradialvelocity/`, `radarreflectivity/`),
  marine (`marine/`), aerosols/composition (`aerosols/`, `insitupm/`),
  composites and wrappers (`compositeoper/`, `categoricaloper/`,
  `timeoper/`, `product/`, `logarithm/`, `pathsum/`), and more.
  Conventional obs types (sondes, aircraft, surface) are served by the
  generic operators plus YAML config.
- **QC filters** (`filters/`) — background/bounds/domain checks, thinning
  (Gaussian, Poisson-disk, temporal, duplicate), Met Office buddy and
  track checks, profile checks, history checks, variable assignment, etc.
- **Obs functions** (`filters/obsfunctions/`) — ~200 files of derived
  quantities usable in filter `where`/assign expressions. Recent
  additions (2026-05/06): `CircularDifference`, `TimeBinner`,
  `ProfileVerticalSmoothing`, `Statistic` (global stats across MPI
  ranks), `LinearTimeInterpolate`.
- **Bias correction** (`predictors/` + `ObsBias*`) — the variational
  bias-correction infrastructure.
- **Variable transforms** (`variabletransforms/`) — observation-side
  variable conversions parallel to VADER's model-side ones.
- **Profile machinery** (`profile/`) — vertical-profile QC plus the
  `ObsProfileAverage` operator and averaging utilities.
- Also: **sampled locations** (`SampledLocations*`), **error models**
  (`errors/`), **field-of-view** (`fov/`), **super-obing** (`superob/`),
  **obs localization** (`obslocalization/`).

This is the largest and most actively-developed JEDI repo by file count.

## How it fits into the bundle

- **Depends on:** `oops`, `ioda`, atlas, eckit, fckit. Optional radiance
  backends: `crtm` (`find_package(crtm 2.4.x)`, v3 also supported) and/or
  `rttov` (v14 preferred, v12.3 fallback). Optional: `gsw` (enables the
  marine observation operators), `ropp-ufo` (GNSS-RO bending angle via
  ROPP; `BUILD_ROPP=ON` in the bundle), `oasim` (`BUILD_OASIM=ON`).
- **Depended on by:** every model wrapper (fv3-jedi, mpas-jedi, soca,
  coupling) that runs DA against observations, plus `simobs`.
- **Build position:** after oops + ioda + crtm (+ gsw).

## Key directories

- `src/ufo/` — the UFO library.
  - Top-level: `ObsTraits.h`, `ObsOperator.{h,cc}` /
    `ObsOperatorBase.{h,cc}`, `LinearObsOperator{,Base}.{h,cc}`,
    `GeoVaLs.{h,cc}` (+ `GeoVaLs.interface.F90`), `ObsFilter{,s,Base}.*`,
    `QCmanager.{h,cc}`, `ObsBias*` (incl. `ObsBiasCovariance`,
    `ObsBiasIncrement`, `ObsBiasPreconditioner`,
    `(Linear)ObsBiasOperator`), `ObsDiagnostics.{h,cc}`,
    `SampledLocations.{h,cc}`, `AnalyticInit*`.
  - `operators/` — forward operators by family (see list above; each
    family dir holds `Obs<Name>.{h,cc}` + `Obs<Name>TLAD.{h,cc}` and
    often Fortran `*.interface.F90` / `ufo_*_mod.F90` backends).
    `operators/gnssro/` subdivides further (`BendMetOffice/`, `BndNBAM/`,
    `BndROPP1D/`, `BndROPP2D/`, `RefMetOffice/`, `RefNCEP/`, `QC/`).
  - `filters/` — QC filters, plus `actions/`, `obsfunctions/`,
    `gnssroonedvarcheck/`, `refractivityonedvarcheck/`,
    `rttovonedvarcheck/` subdirs.
  - `predictors/` — bias-correction predictors (`Constant`, `LapseRate`,
    `Legendre`, `ScanAngle`, `OrbitalAngle`, `CloudLiquidWater`,
    `Emissivity`, `Thickness`, `InterpolateDataFromFile`, …).
  - `variabletransforms/` — observation-space variable conversions.
  - `obslocalization/` — observation localization for EnKF/LETKF
    (GC99, SOAR, vertical).
  - `errors/` — observation-error models (`ObsErrorDiagonal`,
    `ObsErrorDiagonalInvGamma`, `ObsErrorCrossVarCov`,
    `ObsErrorWithinGroupCov`, `ObsErrorDiffusion`,
    `ObsErrorReconditioner`).
  - `profile/` — vertical-profile QC machinery + `ObsProfileAverage`
    operator.
  - `superob/` — super-obing (`SuperObBase`, `SuperObMeanO`,
    `SuperObMeanOmB`, `SuperObRadar`, `SuperObRadarTemplate`).
  - `fov/` — field-of-view geometry (sample/reduce over FOV).
  - `basis/`, `utils/` — supporting infrastructure (`utils/` has
    distance calculators, interpolators, `OceanConversions/`, …).
  - `instantiateObsFilterFactory.h`, `instantiateObsLocFactory.h` —
    factory registration callers for filters and obs localization.
    (There is **no** central operator factory header — operators
    self-register; see Common tasks.)
- `test/` — extensive test suite. `test/mains/` holds the `Test*.cc`
  drivers, `test/ufo/` the test framework headers, `test/testinput/`
  includes `instrumentTests/` (per-instrument: Sonde, Aircraft_obs,
  abi, …) and `unit_tests/` subtrees, `test/testref/` the references.
- `tools/` — `ufo_cpplint.py` (style checker).
- `docs/` — design notes, including `docs/organization.md` (read this
  for an overview of UFO's internal layout).
- `resources/` — packaged config/resource files (`namemap`).

## Key entry points

- `src/ufo/ObsTraits.h` — the `UfoTraits` struct that defines the OBS
  Traits implementation.
- `src/ufo/ObsOperator.{h,cc}` — the OOPS-facing obs operator wrapper;
  `ObsOperatorBase.{h,cc}` is what concrete operators derive from.
- `src/ufo/LinearObsOperator.{h,cc}` — TL/AD wrapper.
- `src/ufo/GeoVaLs.{h,cc}` — model values interpolated to observation
  locations (the bridge between model and obs space).
- `src/ufo/ObsFilters.{h,cc}` — pipeline of QC filters;
  `src/ufo/QCmanager.{h,cc}` — bookkeeping filter that also absorbed the
  former FinalCheck.
- `src/ufo/ObsBias*` — variational bias correction.
- `src/ufo/operators/<family>/` — pick one to learn the operator
  pattern (`operators/identity/` is the simplest).
- `docs/organization.md` — narrative orientation to the code layout.

## Common tasks

- **Run the UFO test suite** — `ctest -L ufo` in the build tree.
- **Add a new observation operator** —
  1. Create `src/ufo/operators/<family>/Obs<MyOp>.{h,cc}` deriving from
     `ObsOperatorBase` (and a TL/AD partner `Obs<MyOp>TLAD` from
     `LinearObsOperatorBase`).
  2. Register with a static maker in the `.cc`:
     `static ObsOperatorMaker<ObsMyOp> maker_("MyOp");` (and
     `LinearObsOperatorMaker` for the TLAD). Add the files to the
     family's `CMakeLists.txt`.
  3. Add a test YAML and reference output under `test/testinput/` and
     `test/testref/` (data fixtures go to `ufo-data`).
- **Add a new QC filter** — derive from `FilterBase` /
  `ObsFilterBase`, register via `instantiateObsFilterFactory.h`, add a
  test.
- **Add an obs function** — new class in `filters/obsfunctions/`,
  self-registers via `ObsFunctionMaker`; test YAMLs live under
  `test/testinput/unit_tests/filters/obsfunctions/`.
- **Use CRTM vs RTTOV** — selected via the YAML
  `obs operator: name` field (`CRTM` vs `RTTOV`); both backends live
  side-by-side under `operators/`.

## Gotchas

- **RTTOV wind speed+direction fallback had u/v swapped (2026-09,
  ufo#4312):** the fallback that decomposes wind speed + direction
  GeoVaLs into components used `u = w*cos(wdir)`, `v = w*sin(wdir)`.
  `var_sfc_wdir` is the azimuth the wind blows *toward*, clockwise from
  north (the convention `uv_to_wdir` produces and CRTM consumes), so the
  correct decomposition is `u = w*sin`, `v = w*cos`. The old form
  reflected the wind vector across the NE-SW diagonal. Both the `s2m` and
  `near_surface` paths were affected. Any RTTOV run that went through the
  speed+direction fallback before 2026-09-04 has wrong surface winds.
- Filter and obs-localization factories register via
  `instantiate*Factory.h`; operators and obs functions register via
  static makers in their own `.cc`. If a new class isn't found at
  runtime, check the right mechanism for its kind (and that the file is
  in CMakeLists).
- `GeoVaLs` variable names must match what VADER (model side) and
  the operator (obs side) agree on — name drift is a common source of
  silent zero outputs.
- The CRTM and RTTOV operators both require their respective
  coefficient files at runtime; missing or version-mismatched
  coefficients produce confusing errors.
- The repo is large; expect non-trivial build times. `ccache` helps.
- **QC flag API change (2026-05):** The QC flag `ObsDataVector` is now
  passed as a reference rather than a shared pointer throughout the filter
  stack (ufo#4112). All ~150 filter classes were updated; any local
  branches touching filters will need rebasing.
- **gsw cmake fix (2026-05):** `gsw_LIBRARIES` cmake usage was corrected
  (ufo#4135); if your build was working around this, the workaround may
  now conflict.
- **GNSSRO test data downsized (2026-05):** large GNSSRO geoval files
  were replaced with smaller variants in ufo-data (ufo-data#562). Tests
  that referenced the old file sizes will need updating.
- **New `DuplicateThinning` filter (2026-05):**
  `src/ufo/filters/DuplicateThinning.{h,cc}` (ufo#4086). Registered in
  `instantiateObsFilterFactory.h`. Use in YAML as
  `filter: Duplicate Thinning`.
- **New obs functions (2026-05):** `ProfileVerticalSmoothing` (ufo#4032),
  `Statistic` (ufo#4091, global stats across MPI ranks), `TimeBinner`
  (ufo#4048), `CircularDifference` (ufo#4055-era) — all under
  `src/ufo/filters/obsfunctions/`.
- **Inverse-Gamma obs error (2026-05, ufo#4024):**
  `src/ufo/errors/ObsErrorDiagonalInvGamma.{h,cc}` plus an
  inverse-gamma effective-stddev statistic in the `EnsembleStatistics`
  filter.
- **`ObsErrorDiffusion` mesh moved to `update` method (2026-05):**
  Mesh construction previously done at construction is now deferred to
  the `update` call (ufo#4129). Local branches overriding
  `ObsErrorDiffusion` will need rebasing.
- **`ObsSpace::put_db` now requires `dimList` (2026-05):** all call
  sites in ufo were updated (ufo#4126); check local branches that call
  `put_db` directly.
- **FinalCheck filter removed (2026-06, ufo#4137):** `src/ufo/filters/FinalCheck.cc/.h`
  deleted; its functionality is now part of the QCmanager filter
  (`src/ufo/QCmanager.cc/.h`). Update any YAML that references
  `filter: Final Check` — use `QCmanager` instead. Also removed from
  `instantiateObsFilterFactory.h`.
- **LinearTimeInterpolate obsfunction added (2026-06, ufo#4049):**
  `src/ufo/filters/obsfunctions/LinearTimeInterpolate.cc/.h` — piecewise
  linear time interpolation/extrapolation between two obs records. Test YAML:
  `test/testinput/unit_tests/filters/obsfunctions/function_lineartimeinterpolate.yaml`.
  Test data is in `ufo-data` (two new `.nc4` files).
- **CRTM coefficient path propagation (2026-06, ufo#4123):** the downloaded
  CRTM coefficient path is now passed through to the relevant UFO modules so
  they skip a redundant download. Works in concert with crtm#302 (CRTM
  exports a `CRTM_TESTFILES_PATH` global property).
- **OSDF btfromradiance fix (2026-06, ufo#4157):** `Cal_SatBrightnessTempFromRad.cc`
  and related variable-transform files fixed to work with OSDF-based
  ObsSpaces.
- **VariableAssignment type checking (2026-06, ufo#4170):**
  `src/ufo/filters/VariableAssignment.cc` now checks the data type of the
  assigned variable and throws a clearer error (including the variable
  name) on mismatch. YAMLs that silently relied on type coercion may now
  fail with an explicit exception.

## Further reading

- In-repo: `README.md`, `docs/organization.md`.
- jedi-docs.jcsda.org → JEDI Components → UFO.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/ioda.md`,
  `jedi-knowledge/crtm.md`, `jedi-knowledge/rttov.md`,
  `jedi-knowledge/ropp-ufo.md`, `jedi-knowledge/gsw.md`,
  `jedi-knowledge/ufo-data.md`, `jedi-knowledge/iodaconv.md`,
  `jedi-knowledge/jedi-bundle.md`.
