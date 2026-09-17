# saber

**Repository:** https://github.com/jcsda-internal/saber
**Branch tracked by bundle:** develop
**Role in JEDI:** Background-error covariance (B-matrix) toolkit for JEDI. Implements the parametric, ensemble, and hybrid B's used by every variational/EnVar application — BUMP, spectral B (SPECTRALB), FastLAM, diffusion, GSI-bec interface, plus block-chain machinery to compose them.

## What it is

SABER (System-Agnostic Background Error Representation) provides a library
of "saber blocks" — outer transforms (e.g. spectral-to-grid, vertical
balance, ψχ→u,v, variable change via VADER) and central blocks (e.g.
parametric correlations, ensemble localization) — that compose into a full
B operator. JEDI's `ErrorCovariance` and `Localization` factories are
populated from SABER, and downstream models pull the resulting B
implementations through OOPS.

SABER bundles a self-contained toy model called QUENCH (in `quench/`) that
mirrors the OOPS `Traits` interface and is used to exercise B
configurations end-to-end (covariance training, randomization, dirac
response) without needing a full atmospheric model.

## How it fits into the bundle

- **Depends on:** `oops 1.10.0`, `vader 1.7.0`, atlas, eckit, fckit,
  NetCDF, MPI, LAPACK/MKL. Optional: `ECTRANS`, `FFTW`, `ip` (NCEP),
  `gsibec` (GSI background error), Torch (for the `torchbalance` block).
- **Depended on by:** every JEDI model that uses a non-trivial B —
  fv3-jedi, mpas-jedi, soca, qg(coupled). They `find_package(saber ...)`
  and link against the appropriate saber libraries.
- **Build position:** after oops and vader.

## Key directories

- `src/saber/` — the saber library (split per block family).
  - `oops/` — glue to OOPS: `ErrorCovariance.h`, `Localization.h`,
    `ErrorCovarianceParameters.h`, `ErrorCovarianceToolbox.h`,
    `ProcessPerts.h`, plus the `instantiateCovarFactory.h` /
    `instantiateLocalizationFactory.h` that downstream code calls to
    register blocks.
  - `blocks/` — the block-chain framework: `BlockChainBase`,
    `OuterBlockBase`, `CentralBlockBase`, `CentralBlockWrapper`
    (multivariate-strategy handling),
    `BlockParametersBase`, `OuterBlockChain`,
    `EnsembleBlockChain`, `HybridBlockChain` (header-only),
    `ParametricBlockChain`, `instantiateBlockChainFactory.h`.
    Note these classes dropped their `Saber` prefix in 2026-09 — see
    **Gotchas**.
  - `bump/` — the BUMP library (Background error on an Unstructured
    Mesh, Benjamin Ménétrier). Core blocks: `NICAS` (correlation),
    `StdDev`, `VerticalBalance`, `PsiChiToUV`, `NICASFilter`. Heavy use
    of fypp templating (`*.fypp`).
  - `spectralb/` — spectral B-matrix blocks: `SpectralCovariance`,
    `SpectralCorrelation`, `SqrtOfSpectralCorrelation`,
    `SqrtOfSpectralCovariance`, `SpectralToGauss`, `SpectralToSpectral`,
    `SpectralAnalyticalCorrelation`, `SpectralAnalyticalFilter`,
    `GaussUVToGP`, `GaussUVToGPWithSMV`, `HydrostaticPressure(m1)`.
  - `fastlam/` — FastLAM (limited-area model fast B): `FastLAM` plus
    `LayerBase`/`LayerHalo`/`LayerRC`/`LayerSpec` variants (requires
    FFTW or ECTRANS).
  - `bifourier/` — bi-Fourier transforms for LAM grids:
    `BifourierCovariance`/`Balance`/`GridToSpectral`, AROME-specific
    blocks (`BifourierAromeBalance`, `BifourierAromeCovariance`,
    legacy interface), FFTW and ECTRANS transform backends.
  - `diffusion/` — diffusion-based correlations (`Diffusion`,
    `DiffusionFilter`, `DiffusionImpl`).
  - `interpolation/` — `AtlasInterpWrapper`, `Interpolation`,
    `GaussToCS` (Gaussian to cubed-sphere), `GaussToCSWithSMV`,
    `SMVInterpWrapper`, `Rescaling`, `VertProj`, `VectorFieldMetadata`,
    `Geometry`.
  - `gsi/` — GSI-bec interface (`GSIBlockChain`, `covariance/`, `grid/`,
    `utils/`); requires `gsibec`.
  - `vader/` — VADER-backed blocks: the generic `VaderBlock` wrapper
    (with its own `DefaultCookbook.h`) plus a large family of
    moisture/balance blocks (`MoistIncrOp`, `SuperMoistIncrOp`,
    `DryMoistIncrOp`, `MoistureControl`, `HydroBal(m1)`,
    `DryAirDensity`, `GpToHp(m1)`, `HpToHexner`, `AirTemperature`,
    ...) and `WriteVariances` diagnostics.
  - `coupled/` — `CoupledErrorCovariance` for multi-component coupled DA.
  - `generic/` — `ID` (identity), `StdDev`, `DuplicateVariables`,
    `OrographicInterp`, `ResidualFields`, `ShadowLevels`, `VertLoc`,
    `VertLocInterp`, `WriteFields`.
  - `torchbalance/` — Torch-based learned balance operator (optional,
    requires Torch_ROOT).
  - `util/` — calibration, binned-fieldset helpers, NetCDF/coordinate
    Fortran helpers.
- `quench/` — toy model.
  - `quench/src/` — `Geometry`, `State`, `Increment`, `Fields`,
    `Covariance`, `VariableChange`, `LinearVariableChange`,
    `Interpolation`, `ModelData`, `Traits.h`, plus a `FieldsIO/`
    subsystem (default and Gmsh backends).
  - `quench/mains/` — `quenchErrorCovarianceToolbox.cc`,
    `quenchConvertState.cc`, `quenchProcessPerts.cc`,
    `quenchSubCommErrorCovarianceToolbox.cc`,
    `quenchCoupledErrorCovarianceToolbox.cc`.
- `test/` — tests with per-block YAML inputs.
  - `test/testinput/` — hundreds of YAMLs (`dirac_*`, `convertcov_*`,
    `compare_diagnostics_*`).
  - `test/testlist/` — per-feature CMake test lists (`saber_bump.cmake`,
    `saber_spectralb.cmake`, `saber_fastlam.cmake`, `saber_gsi-*.cmake`,
    `saber_torchbalance.cmake`, ...).
  - `test/testref/` — reference outputs.
  - `test/fctest/` — Fortran-side tests.
- `tools/` — `saber_plot`, `saber_plot.py`,
  `saber_correlation_variance.py`, `saber_compare_dirac_diagnostics.py`,
  `saber_fit_function.py`, `saber_tidy_yamls.py`, `saber_gcov.sh`,
  `saber_vtune.sh`, `saber_cpplint.py`.
- `docs/` — Doxygen + `tutorial/`, `bump/`, `yaml/` (curated example
  YAMLs), `history/`.

## Key entry points

- `src/saber/oops/ErrorCovariance.h` — the OOPS-facing
  `ErrorCovariance<MODEL>` implementation. The thread that ties saber
  blocks into a JEDI Variational application.
- `src/saber/oops/instantiateCovarFactory.h` — the registration call
  models add to register all SABER blocks with OOPS.
- `src/saber/blocks/OuterBlockBase.h`, `CentralBlockBase.h` —
  base classes for adding a new block.
- `src/saber/blocks/OuterBlockChain.cc`,
  `ParametricBlockChain.cc`, `HybridBlockChain.h` — how
  blocks compose into a full B.
- `src/saber/bump/BUMP.h`, `NICAS.h`, `StdDev.h` — the most widely used
  parametric covariance blocks.
- `quench/mains/quenchErrorCovarianceToolbox.cc` — the canonical
  executable for B training/testing without a real model.
- `docs/yaml/error_covariance_training_tutorial_*.yaml` — annotated
  tutorial YAMLs.

## Common tasks

- **Run tests for one block family** — `ctest -L saber -R bump`, or
  `ctest -R saber_spectralb_`, etc. Test sets are gated on detected
  optional deps (FFTW, ECTRANS, gsibec, Torch).
- **Train a B with QUENCH** — build the QUENCH executables
  (`-DENABLE_QUENCH=ON`, default), then run
  `quenchErrorCovarianceToolbox <yaml>`. Tutorial YAMLs are in
  `docs/yaml/`.
- **Add a new outer block** — derive from `OuterBlockBase`,
  register with a static maker, e.g.
  `static OuterBlockMaker<MyBlock> maker_("my block name");`
  (template: `src/saber/generic/ID.cc`), drop sources into a subdir of
  `src/saber/`, list them in that subdir's `CMakeLists.txt`.
- **Add a new central block** — same pattern with
  `CentralBlockBase` and `CentralBlockMaker<>` (template:
  `src/saber/spectralb/SpectralCovariance.cc`).
- **Plot dirac responses** — `tools/saber_plot.py` /
  `tools/saber_plot/`.
- **Check covariance vs reference** —
  `tools/saber_compare_dirac_diagnostics.py`.
- **Run cpplint** — `ctest -R saber_coding_norms_src`.

## Gotchas

- **SABER classes renamed (2026-09, saber#1288):** the `Saber` prefix
  was dropped from class and file names, since they already live in
  `namespace saber`. `SaberOuterBlockBase` → `OuterBlockBase`,
  `SaberEnsembleBlockChain` → `EnsembleBlockChain`,
  `SaberHybridBlockChain` → `HybridBlockChain`,
  `SaberParametricBlockChain` → `ParametricBlockChain`, and notably
  `SaberCentralBlock` → **`CentralBlockWrapper`** (not
  `CentralBlock`). Files were renamed to match. The full mapping is
  checked in at `tools/rename_classes.py`. **YAML is unaffected** —
  `saber central block`, `saber outer blocks`, `multivariate strategy`,
  `scales` and `multiscale strategy` all keep their names. The rename
  also landed in soca (soca#1258). Any notes or review comments citing
  SABER paths from before 2026-09-10 point at files that no longer exist.
- **`multivariate strategy` has five values, not four.** `single` is the
  **default** (`blocks/CentralBlockWrapper.h`), alongside `univariate`,
  `duplicated`, `duplicated and weighted` and `crossed` (validated in
  `CentralBlockWrapper.cc`). `single` requires a single variable group;
  it errors if `groups` has more than one entry.
- **New `GeographicalMask` outer block (2026-09, saber#1272):**
  `src/saber/generic/GeographicalMask.{h,cc}`, registered as
  `saber block name: GeographicalMask`. Configured with a `masks` list
  (each entry taking `suffix`, `relational operator`, `threshold`), an
  optional `mask variable`, and `read` / `calibration` sections for
  mask and weight file I/O.
- **ShadowLevels refactored (2026-09, saber#1271):** `generic/ShadowLevels`
  reworked and its setup simplified. Test YAMLs renamed
  `dirac_shadowlevels_N` → `dirac_shadow_levels_N` and the old testrefs
  deleted, so branches carrying the old names will fail.
- **Adapted to the new OOPS `GeometryData` (2026-09, saber#1303):**
  follows oops#3333, which moved the interpolator caches out of
  `GeometryData`. See `jedi-knowledge/oops.md`.
- Many block families are conditional on optional deps. Check the
  configure output for "SABER block X is enabled / NOT enabled" lines
  (printed in top-level `CMakeLists.txt`).
- BUMP uses `fypp` preprocessing (`*.fypp`); editing those requires
  `fypp` and a re-cmake. Generated `.F90` files live in the build tree.
- Atlas version gates several features: `ATLAS_REGIONAL_INTERP` for
  Atlas > 0.38.1, `ATLAS_MAKE_SPARSE` for > 0.41, spherical-vector
  interp meta for ≥ 0.44, `ATLAS_SCOPE_ISSUE_RESOLVED` for ≥ 0.46.
  Surprises with mismatched Atlas versions usually trace back here.
- Tests require `jedi-model-data` (>= 1.0.0); without it the `test/`
  subdirectory is skipped silently.
- `quench` is built by default — turn off with `-DENABLE_QUENCH=OFF`
  if you don't have jedicmake or want a faster build. BUMP is likewise
  toggleable (`-DENABLE_BUMP=ON` by default); Torch support is opt-in
  (`-DENABLE_TORCH=OFF` by default).
- BUMP instrumentation is a separate compile flag
  (`-DENABLE_BUMP_INSTRUMENTATION=ON`) used for performance studies.
- **Implicit vertical diffusion added (2026-05):** `DiffusionImpl.cc`
  gained an implicit-scheme option (saber#1234), mirroring oops#3275 and
  UFO's `ObsErrorDiffusion` update. New test YAMLs:
  `test/testinput/dirac_diffusion_implicit_vt.yaml` and
  `error_covariance_training_diffusion_implicit_vt.yaml`.
- **GSI-bec now supports regional fv3jedi and mpasjedi (2026-05):**
  `src/saber/gsi/covariance/gsi_covariance_mod.f90` and
  `src/saber/gsi/grid/gsi_grid_mod.f90` were extended (saber#1088).
  Requires `gsibec`; the GSI-bec test lists (`saber_gsi-geos.cmake`,
  `saber_gsi-gfs.cmake`) have new test entries.
- **Halo-point fix at limited-domain lateral boundaries (2026-05):**
  `src/saber/interpolation/Geometry.cc` — missing halo points along
  the edges of limited domains were previously handled incorrectly
  (saber#1189). Affects FastLAM and regional-domain B configurations.
- **FastLAM improvements (2026-06, saber#1248):** `src/saber/fastlam/`
  — `FastLAM.cc/.h`, `LayerBase`, `LayerHalo`, `LayerRC`, `LayerSpec`
  all updated. If you have local FastLAM patches, expect significant conflicts.
- **SMV Interpolator blocks (2026-06, saber#1241):** New blocks
  `GaussToCSWithSMV` and `SMVInterpWrapper` under `src/saber/interpolation/`,
  and `GaussUVToGPWithSMV` under `src/saber/spectralb/`. Extensive new test
  YAMLs added (`dirac_*_with_smv*.yaml`, `dirac_gaussuvtogp_with_smv.yaml`,
  etc.) with matching `.ref` files under `test/testref/`.
- **ResidualFields block (2026-06, saber#1244):** new
  `src/saber/generic/ResidualFields.cc/.h` — a residual block for filtering
  and a `DryMoistIncrOp` saber block (`src/saber/vader/DryMoistIncrOp.cc/.h`).
- **Scaled ensemble perturbations (2026-06, saber#1240):** user-specified
  scaling options in `src/saber/util/Randomization.cc/.h`. New test:
  `test/testinput/process_perts_spectralb_from_gauss_perts_6.yaml` /
  `test/testref/process_perts_spectralb_from_gauss_perts_6.ref`.
- **Custom partitioner bugfixes (2026-06, saber#1250):** fixes in block-chain
  partitioning logic in `src/saber/blocks/EnsembleBlockChain` and
  `ParametricBlockChain`.
- **Torch/Python virtual environment unified (2026-06, saber#1207):**
  the torchbalance block and Python tools now share the same virtual
  environment; CI configuration in `.github/workflows/ci.yml` updated.

## Further reading

- In-repo: `README.md`, `docs/tutorial/`, `docs/yaml/` (annotated YAMLs),
  `docs/bump/plot.py`.
- jedi-docs.jcsda.org: SABER section, plus the BUMP and SPECTRALB
  tutorials.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/vader.md`,
  `jedi-knowledge/fv3-jedi.md`, `jedi-knowledge/mpas-jedi.md`,
  `jedi-knowledge/soca.md`, `jedi-knowledge/jedi-model-data.md`,
  `jedi-knowledge/coupling.md`.
