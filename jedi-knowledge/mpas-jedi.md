# mpas-jedi

**Repository:** https://github.com/jcsda-internal/mpas-jedi
**Branch tracked by bundle:** develop
**Role in JEDI:** the MPAS-side OOPS Traits implementation — adapts the MPAS-Atmosphere model to JEDI so it can be driven by oops applications (Variational, HofX, Forecast, EnKF, …).

## What it is

`mpas-jedi` (project name `mpasjedi` in CMake) is the model interface
that wraps NCAR/LANL's MPAS-Atmosphere model for JEDI. Like `fv3-jedi`
and `soca`, it implements the OOPS Traits surface so generic
`oops::` algorithms run on MPAS configurations. Version 3.1.0.

CMake requires `MPAS 8.0` with the `core_atmosphere` core (plus
`DOUBLE_PRECISION` and `OpenMP` components when those are enabled).
Precision follows the `MPAS_DOUBLE_PRECISION` flag set by the bundle
(default ON).

## How it fits into the bundle

- **Depends on:** the cloned `mpas` repo (MPAS-Model with at least
  `core_atmosphere`), `oops`, `ufo`, `saber`, `vader`, `crtm`, atlas
  (>= 0.35.0).
- **Depended on by:** user driver code; not consumed by other bundle
  components.
- **Build position:** after MPAS-Model and the JEDI framework;
  `mpas-jedi-data` is declared before it in the bundle.

## Key directories

- `src/mpasjedi/` — the MPAS-JEDI library, organized into one subdir
  per Traits component plus internals:
  `Covariance/`, `Fields/`, `Geometry/`, `GeometryIterator/`,
  `Increment/`, `LinearVariableChange/`, `Model/`, `ModelBias/`,
  `ModelData/`, `Saca/`, `State/`, `Tlm/`, `Utilities/`,
  `VariableChange/`. Each interface subdir typically holds the C++
  class (`X.h/.cc`), a `X.interface.h`, and the Fortran implementation
  (`mpas_x_mod.F90`, `mpas_x_interface.F90`). The heavy Fortran field
  logic is in `src/mpasjedi/Fields/mpas_fields_mod.F90`.
- `src/mains/` — application drivers (15): `mpasVariational.cc`,
  `mpasHofX.cc`, `mpasHofX3D.cc`, `mpasForecast.cc`, `mpasEDA.cc`,
  `mpasEnKF.cc`, `mpasEnsHofX.cc`, `mpasEnsMeanVariance.cc`,
  `mpasErrorCovarianceToolbox.cc`, `mpasGenEnsPertB.cc`,
  `mpasProcessPerts.cc`, `mpasRTPP.cc`, `mpasSACA.cc`,
  `mpasConvertState.cc`, `mpasConvertToStructuredGrid.cc`.
  Executables are named `mpasjedi_<app>.x` (e.g.
  `mpasjedi_variational.x`, `mpasjedi_hofx3d.x`,
  `mpasjedi_error_covariance_toolbox.x`) — see
  `src/mains/CMakeLists.txt`.
- `test/` — `testinput/` (~71 test YAMLs plus
  `testinput/namelists/384km/` and `testinput/namelists/480km/` MPAS
  namelists, including `*_regional` variants), `testoutput/` (`.ref`
  reference files), `executables/` (`Test*.cc` unit-test drivers for
  Geometry, State, Increment, Model, TLM, ErrorCovariance, etc.),
  `covariance/` (offline B-matrix processes, yamls, plotting).
- `graphics/` — Python diagnostics package (`AnalyzeStats.py`,
  `DiagnoseObsStatistics.py`, plotting utilities) for OMB/OMA and
  model-space statistics.
- `workflow/` — csh scripts (`da.csh`, `drive.csh`, `fc1.csh`, …) for
  running cycling experiments outside of EWOK (legacy / standalone).
- `CI/` — AWS CodeBuild specs (clang/gnu/intel).

## Key entry points

- `src/mpasjedi/Traits.h` — the `mpas::Traits` struct
  (`name() == "MPAS"`); maps OOPS types to the mpas implementations;
  ObsLocalization comes from `ufo::ObsLocalization`.
- `src/mains/mpasVariational.cc` — canonical 3DVar / EnVar / 4DVar
  driver for MPAS.
- `src/mains/mpasHofX.cc` / `mpasHofX3D.cc` — HofX (forward operator)
  drivers.
- `src/mpasjedi/Geometry/Geometry.{h,cc}` + `mpas_geom_mod.F90` —
  wraps the MPAS unstructured mesh for Atlas / OOPS;
  `GeometryParameters.h` holds geometry YAML options (including
  `"regional nn fill distance in km"`).
- `src/mpasjedi/Model/Model.{h,cc}` + `mpas_model_mod.F90` — MPAS
  forecast model wrapper.
- `src/mpasjedi/State/mpas_state_mod.F90` — state read/write and
  analysis update logic.

## Common tasks

- **Run the MPAS-JEDI reference tests** — `ctest -R mpasjedi_` in the
  build tree (data fixture comes from `mpas-jedi-data`).
- **Run a 3DEnVar experiment** — driven by skylab + ewok; executable
  is `mpasjedi_variational.x` invoked with a YAML.
- **Switch precision** — controlled by `MPAS_DOUBLE_PRECISION`
  (must match the MPAS-Model build; the bundle sets it for both).
- **Add a new variable change** — drop a class under
  `src/mpasjedi/VariableChange/`, register, add a test YAML.

## Gotchas

- **Tropopause pressure method is configurable again (2026-09,
  mpas-jedi#1238):** `mpasjedi_vc_model2geovars_mod.F90` had the method
  hardcoded to `thompson` with the YAML lookup commented out. The
  `tropopause pressure method` key is read once more and now accepts
  `thompson` (default) or `wmo`; the WMO branch was un-commented.
- `mpas-jedi` requires the bundle's `mpas` repo built with the right
  core(s) and matching precision. The current 2-stream I/O test data
  was generated with a **single-precision** MPAS-Model run
  (mpas-jedi#1181 / mpas-jedi-data#14); `.ref` files on develop form a
  single set — mismatched precision between your MPAS build and the
  refs can show up as numeric ctest diffs. (A
  `feature/mpas_single_precision_testing` branch exists upstream for
  precision-specific reference outputs but is not on develop.)
- The standalone tarball fallback for test data is pinned to
  `3.1.0.jcsda` in `mpas-jedi/CMakeLists.txt` (used when the
  `mpas-jedi-data` repo isn't found; logic in `test/CMakeLists.txt`).
  Bumping mpas-jedi may require also bumping the tarball pin.
- The `workflow/` csh scripts are *legacy* — for production
  experiments use skylab + ewok instead.
- **Regional mesh ctests (2026-05):** `3denvar_multi_resolution_regional`
  (384 km mesh) and `parameters_bumploc_regional` (480 km mesh with
  ensemble) require the regional test data in `mpas-jedi-data`
  (`conus384km.*` / `conus480km.*` files). Test namelists live in
  `test/testinput/namelists/384km/` and
  `test/testinput/namelists/480km/` (including `*_regional` variants).
- **New `"regional nn fill distance in km"` geometry parameter
  (2026-05):** added in `src/mpasjedi/Geometry/GeometryParameters.h`
  and the Fortran geom module (mpas-jedi#1199). Controls nearest-
  neighbour fill distance for regional mesh geometries.
- **`smoke_fine` added to `field_is_scalar` logic (2026-05):**
  `src/mpasjedi/Fields/mpas_fields_mod.F90` — affects which fields are
  treated as scalars in the DA path (mpas-jedi#1136, follow-up fix
  #1194 for an illegal Fortran array spec).
- **2m T,q now updated between outer loops (2026-05):**
  `src/mpasjedi/State/mpas_state_mod.F90` (mpas-jedi#1174). Affects
  multi-outer-loop experiments; reference outputs were updated.
- **CO2 absorber for CrIS & IASI (2026-06, mpas-jedi#1203):** the CRTM
  operator config now includes CO2 as an absorber for CrIS and IASI
  channels. Affects HofX and radiance assimilation tests — related refs
  were updated.
- **CDA ctest expansion (2026-06, mpas-jedi#1146):** `3dfgat_cda.yaml`,
  `4dfgat_cda.yaml`, and associated ref files were updated to exercise
  new CDA capabilities. `3dfgat_cda` also gained `tier2` labels
  (mpas-jedi#1205).
- **Integer field interpolation type corrected (2026-06,
  mpas-jedi#1200):** `src/mpasjedi/Fields/mpas_fields_mod.F90`.
- **Broad ref-output update (2026-06, mpas-jedi#1204):** 55 files
  changed — letkf, lgetkf, parameters_bumpcov/bumploc,
  process_perts_spectral, rtpp, 3denvar, 4denvar, eda, forecast, hofx,
  dirac, gen_ens_pert_B refs all regenerated. Any branch with local
  edits to these files should expect heavy merge conflicts.
- **oops safety-check removal follow-up (2026-06, mpas-jedi#1207):**
  develop HEAD tracks an oops API change (removal of a safety check) —
  building an older mpas-jedi against current oops (or vice versa) can
  break; keep the bundle repos in sync.

## Further reading

- jedi-docs.jcsda.org → JEDI Components → MPAS-JEDI.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/ufo.md`,
  `jedi-knowledge/saber.md`, `jedi-knowledge/vader.md`,
  `jedi-knowledge/crtm.md`, `jedi-knowledge/mpas.md`,
  `jedi-knowledge/mpas-jedi-data.md`, `jedi-knowledge/jedi-bundle.md`.
