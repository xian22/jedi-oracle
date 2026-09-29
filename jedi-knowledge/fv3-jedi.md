# fv3-jedi

**Repository:** https://github.com/jcsda-internal/fv3-jedi
**Branch tracked by bundle:** develop
**Role in JEDI:** the FV3-side OOPS Traits implementation — adapts FV3-based atmospheric models (FV3-GFS, GEOS, UFS) to JEDI so they can be driven by oops applications (Variational, HofX, Forecast, EnKF, …).

## What it is

`fv3-jedi` (project version 1.9.0) is the model interface that wraps the
FV3 family of dynamical cores (NOAA FV3-GFS, NASA GEOS, NOAA UFS) so
JEDI's generic `oops::` algorithms can drive them. The trait struct is
`fv3jedi::Traits` (defined in `src/fv3jedi/Utilities/Traits.h`) and it
binds: `Geometry`, `GeometryIterator`, `State`, `Increment`,
`ErrorCovariance` (as `Covariance`), `ModelWrapper` (as `Model`), `Tlm`
(as `LinearModel`, backed by fv3-jedi-lm), `VariableChange`,
`LinearVariableChange`, `ModelBias`/`ModelBiasIncrement`/
`ModelBiasCovariance` (the `ModelAux*` types), `ModelData`,
`NormGradient`, `oops::UnstructuredInterpolator` (LocalInterpolator),
and `ufo::ObsLocalization<GeometryIterator>`.

CMake options choose which FV3 forecast model to build the interface
for: `FV3_FORECAST_MODEL` (FV3CORE / GEOS / UFS), and
`UFS_APP` (ATM / ATMAERO / S2S) when UFS is selected. UFS builds are
now done only via the ufs-bundle, which builds the UFS as an external
library (see the comment in `CMakeLists.txt`); UFS ctests were removed
from this repo (#1488).

## How it fits into the bundle

- **Required dependencies:** `oops >= 1.10`, `ufo >= 1.10`,
  `saber >= 1.10`, `vader >= 1.7`, `atlas >= 0.35`,
  `FMS >= 2023.04` (R4+R8 components), `fv3jedilm >= 1.5`
  (the linear model from `fv3-jedi-lm`), NetCDF, MPI.
- **Optional dependencies (found QUIET):** `crtm >= 2.2.3` (optional
  since #1506 — when absent, CRTM-dependent tests are disabled),
  `ropp-ufo`, `gsibec >= 1.4.2` (enables the FV3-SABER GSI block via
  `-DENABLE_GSIBEC=1`), `ip` (NCEP interpolation), `jedicmake`.
- **Depended on by:** `coupling` (combines fv3-jedi + soca for
  atm-ocean DA) and any user driver code that links FV3 with JEDI.
- **Build position:** late in the bundle (after oops, ufo, saber,
  vader, fv3-jedi-lm, and the FMS chosen by the bundle).

## Key directories

- `src/fv3jedi/` — one subdirectory per OOPS Traits component:
  `Geometry/` (plus `Geometry/fv3/`), `State/`, `Increment/`,
  `Fields/` (Fortran-side `fv3jedi_field_mod.f90` /
  `fv3jedi_fields_mod.f90` underlying State and Increment),
  `FieldMetadata/`, `Model/` (`ModelBase`/`ModelWrapper` plus
  `fv3lm/`, `geos/`, `ufs/`, `pseudo/`), `Tlm/`,
  `IO/` (`FV3Restart/`, `CubeSphereHistory/`, `StructuredGrid/`,
  plus `Utils/` with the `IOBase` factory), `VariableChange/`
  (`Base/`, `Control2Analysis/`, `Model2GeoVaLs/`, `Utils/`, `femps/`,
  `VaderCookbook.h`), `LinearVariableChange/` (`Base/`,
  `Control2Analysis/`, `Model2GeoVaLs/`), `ErrorCovariance/`,
  `ModelBias/`, `NormGradient/`, `ObsLocalization/`
  (`ObsLocVerticalBrasnett.h`), `GeometryIterator/`, `ModelData/`,
  `Utilities/` (Traits.h, constants, FMS namelist helpers,
  `fv3jedi_vertical_remap.{h,cc}`).
- `src/mains/` — 23 C++ and 4 Fortran application drivers producing
  ~26 executables (e.g. `fv3jedi_var.x`, `fv3jedi_hofx.x`,
  `fv3jedi_hofx_nomodel.x`, `fv3jedi_forecast.x`,
  `fv3jedi_adjointforecast.x`, `fv3jedi_eda.x`, `fv3jedi_letkf.x`,
  `fv3jedi_EnsGETKF.x`, `fv3jedi_convertstate.x`,
  `fv3jedi_convertincrement.x`, `fv3jedi_converttostructuredgrid.x`,
  `fv3jedi_genenspertB.x`, `fv3jedi_error_covariance_toolbox.x`,
  `fv3jedi_tlm_toolbox.x`, `fv3jedi_controlpert.x`,
  `fv3jedi_processperts.x`, `fv3jedi_ensmeanvariance.x`,
  `fv3jedi_ensrecenter.x`, `fv3jedi_enshofx.x`,
  `fv3jedi_gen_hybrid_linear_model_coeffs.x`, `fv3jedi_diffstates.x`,
  `fv3jedi_addincrement.x`; Fortran utilities `fv3jedi_a_to_d.x`,
  `fv3jedi_d_to_a.x`, `fv3jedi_cold_to_warm.x`,
  `fv3jedi_geos_to_fms.x`).
- `test/` — ~194 test YAMLs in `testinput/` and ~146 reference outputs
  in `testoutput/` covering 3DVar, 4DVar, EnVar, HofX, LETKF/GETKF,
  ConvertState, IO round-trips, and variable-change checks; `Data/`
  (fv3files, GeosFiles, gsibec namelists, forecast fixtures) and
  helper scripts (`setup-c48.sh`, `prep-letkf.sh`,
  `fv3jedi_setup_ufs_rundir.py`). Test tiers are expressed as ctest
  LABELS (e.g. `tier2`); the old `FV3JEDI_TEST_TIER` variable was
  removed (#1482).
- `Documents/` — Doxygen config; built only with
  `-DENABLE_FV3JEDI_DOC=ON`.

## Key entry points

- `src/fv3jedi/Utilities/Traits.h` — the `fv3jedi::Traits` struct that
  ties all the FV3 component implementations together for OOPS
  templates.
- `src/mains/fv3jediVar.cc`, `fv3jediHofX.cc` (instantiates
  `oops::HofX4D`), `fv3jediForecast.cc`, `fv3jediEDA.cc`,
  `fv3jediLETKF.cc` — canonical executable templates.
- `src/fv3jedi/Geometry/Geometry.{h,cc}` — start here to see how an
  FV3 cubed-sphere grid is wrapped for JEDI/Atlas.
- `src/fv3jedi/State/State.{h,cc}`, `Increment/Increment.{h,cc}`,
  `Fields/fv3jedi_fields_mod.f90` — the core state representation
  (C++ wrappers over Fortran field storage).
- `src/fv3jedi/Model/fv3lm/`, `Model/geos/`, `Model/ufs/`,
  `Model/pseudo/` — per-flavor forecast model wrappers; the flavor is
  selected by `FV3_FORECAST_MODEL` (pseudo reads pre-computed states).
- `src/fv3jedi/IO/FV3Restart/`, `CubeSphereHistory/`,
  `StructuredGrid/` — the three I/O backends, dispatched through
  `IO/Utils/IOBase.{h,cc}`. FV3Restart is FMS2-IO only
  (`fv3jedi_io_fms2_mod.f90`).
- `src/fv3jedi/FieldMetadata/FieldsMetadata.{h,cc}` and
  `FieldsMetadataDefault.h` — variable metadata (long names, units,
  vertical levels) consulted by VariableChange, IO, and ModelData.
- `src/fv3jedi/VariableChange/VaderCookbook.h` — the vader recipe
  cookbook used by FV3 variable changes.

## Common tasks

- **Run the FV3-GFS reference tests** — configure with
  `-DFV3_FORECAST_MODEL=FV3CORE` (the default code path), then
  `ctest -R fv3jedi_`.
- **Switch to GEOS** — `-DFV3_FORECAST_MODEL=GEOS` with
  `FV3_FORECAST_MODEL_ROOT` pointing at a GEOS install (MAPL, GEOSgcm
  and friends are then required).
- **Build with UFS** — use the ufs-bundle; `-DFV3_FORECAST_MODEL=UFS
  -DUFS_APP=ATM` (or `ATMAERO` / `S2S`) requires ESMF (`esmf_ROOT`)
  and the NCEP libs (bacio, sigio, sp, w3emc, nemsio).
- **Add a new variable change** — drop a new class under
  `src/fv3jedi/VariableChange/`, register it in the factory
  (`Base/VariableChangeBase.h`), add a test YAML in `test/testinput/`;
  prefer a vader recipe (see `VaderCookbook.h`) when the transform is
  model-agnostic.
- **Run a 3DVar experiment** — typically driven by the workflow stack
  via skylab + ewok; the executable is `fv3jedi_var.x` invoked with a
  YAML.

## Gotchas

- **`f10m` computed when missing (2026-09, fv3-jedi#1551):**
  `model2geovals` now calculates the 10 m wind factor from wind speed if it
  isn't already in the state, guarding against a zero denominator. This
  affects operators that consume near-surface wind GeoVaLs.
- The build is sensitive to FMS / Atlas / OOPS versions — bumping any
  of these often requires fv3-jedi changes too (the `>=` pins are
  conservative, not aspirational).
- Three different forecast models are gated behind one CMake option:
  builds for one are not interchangeable with another's executables.
  Choose `FV3_FORECAST_MODEL` deliberately.
- Test data comes from `fv3-jedi-data` when found
  (`find_package(fv3-jedi-data QUIET)`); otherwise CMake falls back to
  a tarball pinned to tag 1.9.0 (md5-checked) from
  `bin.ssec.wisc.edu`. Bumping fv3-jedi test data means bumping
  `fv3-jedi_data_tag` in `CMakeLists.txt` too.
- 4DVar requires the linear model — `fv3-jedi-lm` must be built and
  found via `find_package(fv3jedilm 1.5.0 REQUIRED)`. It is a hard
  requirement of the build, not an option.
- **CRTM is optional (2026-06, fv3-jedi#1506):** `find_package(crtm
  2.2.3 QUIET)` — fv3-jedi builds succeed without CRTM, and
  CRTM-dependent tests are simply disabled. There is no `WITH_CRTM`
  switch; presence of a findable CRTM is what enables those paths.
- **FMS2-only IO (2026-06, fv3-jedi#1509):** remaining non-FMS2 I/O
  code was removed (`src/fv3jedi/IO/FV3Restart/fv3jedi_io_fms2_mod.f90`
  and the Fortran mains). The CMake minimum is still FMS 2023.04, but
  pair with a recent FMS — fv3-jedi-lm's develop is aligned with
  FMS 2025.2.
- **Level ordering (2026-06, fv3-jedi#1510):** a "levels are top down"
  fix in `src/fv3jedi/ModelData/ModelData.{cc,h}` changed convertstate
  reference outputs (`test/testoutput/convertstate_*.ref`). Branches
  with local ref-output edits will see conflicts there.
- **GSIBEC adjoint test enforced (2026-06, fv3-jedi#1517):** tests using
  the GSI static covariance SABER block now set `adjoint test: true` and
  add an extra tight-tolerance minimization (`gradient norm reduction:
  1e-10`) in their `variational:` section; the GSIBEC namelists under
  `test/Data/gsibec/` and several `test/testoutput/*.ref` files changed
  to match, and `fv3jedi_hyb-4denvar_geos_sondes` /
  `fv3jedi_hyb-4denvar_llens_geos` lost their `tier2` /
  `ci_flake_disable` labels (they now run in the default tier). Branches
  carrying local edits to those test YAMLs or refs will conflict.

## Further reading

- jedi-docs.jcsda.org → JEDI Components → fv3-jedi.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/ufo.md`,
  `jedi-knowledge/saber.md`, `jedi-knowledge/vader.md`,
  `jedi-knowledge/crtm.md`, `jedi-knowledge/fv3-jedi-data.md`,
  `jedi-knowledge/fv3-jedi-lm.md`, `jedi-knowledge/coupling.md`,
  `jedi-knowledge/jedi-bundle.md`.
