# soca

**Repository:** https://github.com/jcsda-internal/soca
**Branch tracked by bundle:** develop (cloned RECURSIVE)
**Role in JEDI:** the ocean / sea-ice OOPS Traits implementation — built on the MOM6 grid (via FMS), with Icepack for sea-ice physics, so JEDI's generic algorithms can drive ocean and sea-ice data assimilation.

## What it is

SOCA is the ocean-DA companion to fv3-jedi/mpas-jedi: a "JEDI
encapsulation of MOM6" (the README's words). Version 1.8.0. It
implements the OOPS Traits surface in `src/soca/Traits.h`:

- `Geometry`, `GeometryIterator`, `State`, `Increment`
- `Covariance` = `soca::ErrorCovariance` (empty stub — real B comes
  from SABER)
- `Model` / `LinearModel` = `ModelOceanIceEmulator` /
  `LinearModelOceanIceEmulator` (despite the name, currently a
  pseudo-model stub — see gotchas; MOM6 itself is **not** wrapped as
  an `oops::Model`)
- `ModelAuxControl/Increment/Covariance` = the `ModelBias` trio
- `VariableChange`, `LinearVariableChange`, `ModelData`
- `LocalInterpolator` = `oops::UnstructuredInterpolator`,
  `ObsLocalization` = `ufo::ObsLocalization`

The bundle clones SOCA `RECURSIVE`: MOM6 (NOAA-EMC/MOM6) and Icepack
(CICE-Consortium/Icepack) are git submodules at `external/mom6/MOM6`
and `external/icepack/Icepack`, built by SOCA's CMake as libraries
`mom6` and `icepack_limited`.

## How it fits into the bundle

- **Depends on (find_package):** `oops` (1.10), `vader` (1.7), `saber`
  (1.10), `ioda` (2.9), `ufo` (1.10), `fms` (2023.3, R8 alias),
  eckit/fckit/atlas, NetCDF, gsl-lite, OpenMP. Also links `gsw`
  directly (marine thermodynamics). Optional: Torch (set `Torch_ROOT`)
  enables the `MLBalance/` SABER block and the `soca_ice_emulator.x`
  training/prediction executable.
- **Depended on by:** `coupling` (combines SOCA with fv3-jedi for
  atm-ocean coupled DA).
- **Build position:** late, after the framework repos and gsw.

## Key directories

- `src/soca/` — the SOCA library. Subdirs: `AnalyticInit/`,
  `Covariance/`, `Fields/`, `Geometry/`, `GeometryIterator/`, `IO/`,
  `Increment/`, `LinearModel/`, `LinearVariableChange/`, `MLBalance/`
  (Torch-only), `Model/`, `ModelBias/`, `ModelData/`,
  `ObsLocalization/`, `SaberBlocks/`, `State/`, `Utils/`,
  `VariableChange/`; plus `Traits.h` and `Fortran.h`.
- `src/mains/` — application drivers (see entry points).
- `external/` — MOM6 + Icepack submodules and the CMake glue
  (`mom6_files.cmake`, `icepack_files.cmake`) that builds them.
- `bundle/CMakeLists.txt` — SOCA's standalone bundle (gsw, oops,
  vader, saber, ioda, ufo, oasim + data repos) for ocean-only builds
  outside the larger jedi-bundle.
- `test/` — ~80 `soca_add_test` entries registered as `test_soca_*`;
  YAMLs in `test/testinput/` (~80 files), references in
  `test/testref/`, fixtures in `test/Data/` (two tri-polar grids:
  `36x17x25` and `72x35x25`, plus obs, oasim, and mlbalance data).
  `test/mom6solo.py` drives free-running MOM6 forecasts with the
  standalone `mom6solo` executable built from `external/mom6`.
- `docs/` — Doxygen config plus `postproc_application.md` (ensemble
  analysis post-processing documentation).
- `cmake/` — compiler-flag modules (`soca_compiler_flags`).

## Key entry points

- `src/soca/Traits.h` — the SOCA Traits struct (see above).
- `src/soca/Geometry/Geometry.{h,cc}` + `soca_geom_mod.F90` — wraps
  the MOM6/FMS grid for Atlas/OOPS; `FmsInput.{h,cc}` handles FMS
  namelist staging.
- `src/soca/Fields/soca_fields_mod.F90` — Fortran field storage; with
  `src/soca/Geometry/soca_geom_mod.F90`, the only places that `use`
  MOM6 modules directly.
- `src/soca/IO/soca_io_mod.F90` — direct NetCDF I/O for state and
  geometry (its own CMake sub-library `soca_IO`).
- `src/soca/VariableChange/` — nonlinear transforms: `Model2Ana`,
  `Model2GeoVaLs`, `Soca2Cice` (writes analysis back to CICE restarts
  via Icepack; has its own README).
- `src/soca/LinearVariableChange/` — `Balance` (KST / KSSHTS balance
  operators), `LinearModel2GeoVaLs`.
- `src/soca/SaberBlocks/` — `BkgErrFilt`, `ParametricOceanStdDev`.
- `src/soca/ObsLocalization/ObsLocRossby.{h,cc}` — Rossby-radius-based
  obs localization.
- `src/soca/Utils/` — `OceanSmoother`, increment QC (`incrqc/`),
  `readNcAndInterp`.
- `src/mains/` — executables: `soca_var.x`, `soca_hofx.x`,
  `soca_hofx3d.x`, `soca_enshofx.x`, `soca_forecast.x`,
  `soca_letkf.x` (LETKF/GETKF/EAKF via oops LocalEnsembleDA),
  `soca_gridgen.x`, `soca_error_covariance_toolbox.x`,
  `soca_enspert.x`, `soca_sqrtvertloc.x`, `soca_ensmeanandvariance.x`,
  `soca_ensrecenter.x`, `soca_hybridgain.x`, `soca_anpproc.x`
  (analysis post-processing, `AnalysisPostproc.{h,cc}`),
  `soca_addincrement.x`, `soca_diffstates.x`, `soca_convertincrement.x`,
  `soca_convertstate.x`, `soca_converttostructuredgrid.x`,
  `soca_setcorscales.x`, `soca_gen_hybrid_linear_model_coeffs.x`,
  `soca_tlm_toolbox.x`, and (Torch-only) `soca_ice_emulator.x`.

## Common tasks

- **Run the SOCA tests** — clone with `--recursive` so MOM6 + Icepack
  come along, build, then `ctest -R test_soca`. The gridgen test
  produces the gridspec the others depend on.
- **Run a 3DVar experiment** — `soca_var.x` with a YAML (see
  `test/testinput/3dvar.yml`, `3dhyb.yml`); production runs are
  typically driven by skylab + ewok.
- **Run a free MOM6 forecast** — JEDI-side forecast apps use generic
  oops models (identity, pseudo); a real MOM6 integration uses the
  standalone `mom6solo` executable (see `test/mom6solo.py` for the
  invocation pattern).
- **Update the MOM6 / Icepack pin** — `git submodule update --remote
  external/mom6/MOM6` (or `external/icepack/Icepack`) and review impact
  on `soca_geom_mod.F90` / `soca_fields_mod.F90` and the
  `external/*/[mom6|icepack]_files.cmake` source lists.

## Gotchas

- **SABER class renames applied (2026-09, soca#1258):** soca's SABER
  blocks were updated for saber#1288, which dropped the `Saber` prefix
  from SABER class names (e.g. `SaberOuterBlockBase` → `OuterBlockBase`).
  Affects `ParametricOceanStdDev` and the other blocks under
  `src/soca/SaberBlocks/`. See `jedi-knowledge/saber.md`.
- **Recursive clone is required.** A non-recursive clone leaves
  `external/` submodules empty and the build fails.
- **There is no MOM6 `oops::Model` wrapper.** The only Model in the
  Traits is `ModelOceanIceEmulator`; real MOM6 forecasts run via the
  standalone `mom6solo` executable outside JEDI. Don't go looking for
  a `src/soca/Model/MOM6` class — it was removed (soca#1026, #1181).
- **`ModelOceanIceEmulator` is not (yet) an emulator.** Despite the
  name, `src/soca/Model/OceanIceEmulator/ModelOceanIceEmulator.cc` is
  a no-op pseudo-model: `step()` only advances the valid time, and
  there is no Torch code in `Model/` or `LinearModel/` (both build
  without Torch). It is scaffolding for a future ML forecast emulator.
  The actual ML/Torch code in SOCA is `MLBalance` — a SABER outer
  block (registered as `"MLBalance"` in `src/soca/MLBalance/MLBalance.cc`)
  whose `KEmul/` FFNN (`BaseEmul.h`, `IceEmul.h`, `IceNet.h`) supplies
  the balance Jacobian; `soca_ice_emulator.x` trains/evaluates that
  FFNN from CICE history files. SABER separately ships a generic
  Torch-based balance block (`saber/src/saber/torchbalance/`) — see
  `jedi-knowledge/saber.md`.
- **Torch is optional but gates features.** Without `Torch_ROOT`,
  `MLBalance/` and `soca_ice_emulator.x` are not built; YAMLs that use
  the ML balance or emulator (e.g. `4dvar_oceaniceemulator.yml`,
  `3dvar_diffmlb.yml`) won't work.
- The two test grids (36x17x25 and 72x35x25) are low-res tri-polar
  fixtures — realistic experiments use user-supplied higher-res
  configs.
- **New direct NetCDF I/O module (2026-05, soca#1242):**
  `src/soca/IO/soca_io_mod.F90` was added as CMake sub-library
  `soca_IO`; `soca_fields_mod.F90` and `soca_geom_mod.F90` were
  refactored to use it. Local branches touching state/geometry I/O
  will see significant merge conflicts.
- **l/getkf test references updated (2026-05, soca#1235):** letkf/getkf
  refs regenerated for the QC change (QC applied to ensemble
  mean/modulated members, oops#3231). Re-run `ctest -R letkf` after
  rebuilding if your build predates this.
- **FMS 2.1 compatibility (2026-06, soca#1245):**
  `src/soca/Geometry/soca_geom_mod.F90` now uses
  `time_interp_external2_mod` instead of the deprecated
  `time_interp_external_mod`. Builds against older FMS may need this
  reverted.
- **Ensemble postproc zero-out list now independent (2026-06,
  soca#1244):** `src/mains/AnalysisPostproc.h` no longer requires the
  zero-out variable list to be a subset of the increment variables.

## Further reading

- jedi-docs.jcsda.org → JEDI Components → SOCA; `docs/postproc_application.md`
  in the repo for the ensemble analysis post-processing application.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/ufo.md`,
  `jedi-knowledge/vader.md`, `jedi-knowledge/saber.md`,
  `jedi-knowledge/coupling.md`, `jedi-knowledge/jedi-bundle.md`.
