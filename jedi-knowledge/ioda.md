# ioda

**Repository:** https://github.com/jcsda-internal/ioda
**Branch tracked by bundle:** develop
**Role in JEDI:** Interface for Observational Data Access — JEDI's observation-data layer. Defines `ioda::ObsSpace` (the in-memory representation of an observation set), the on-disk IODA file conventions, the storage engines (HDF5, ODB, BUFR, in-memory), and the distribution machinery that splits observations across MPI ranks.

## What it is

IODA is the standardized observation I/O and in-memory data layer that
sits between raw observation files and JEDI's computational code. UFO
operates on `ioda::ObsSpace` objects; H(x) and feedback round-trip
through IODA files. The native IODA file format is HDF5/NetCDF4 with
strong group/variable/metadata conventions; ODB and BUFR are also
supported as read (and ODB as write) formats.

The library provides:

- The C++ `ObsSpace` API (the canonical in-memory representation,
  derived from `oops::ObsSpaceBase`), plus `ObsVector` and
  `ObsDataVector`.
- **OSDF** (Observation Space Data Frame) — a newer in-memory container
  layer in `src/containers/` (`FrameCols`, `FrameRows`, `IFrame`
  slicing interface, column metadata). An OSDF-based ObsSpace flow is
  being brought to parity with the classic one.
- **ioda-engines** (`src/engines/ioda/`) — a self-contained subpackage
  with the low-level Group/Variable/Attribute object model and the
  backend engines: HH (HDF5), ObsStore (in-memory), ODC (ODB), Bufr,
  Script, GenList/GenRandom.
- **Distributions** (`src/distribution/`) that partition obs across
  MPI ranks: `Halo`, `RoundRobin`, `InefficientDistribution`,
  `IdentityDistribution`, `NonoverlappingDistribution`,
  `AtlasDistribution`, `PairOfDistributions`, `ReplicaOf*`, plus
  `SelectedRanks` and accumulator helpers.
- **Reader / writer / I/O pool** infrastructure for parallel I/O
  (`src/reader/`, `src/writer/`, `src/ioPool/`, `src/obsIoPool/`).
- Fortran APIs (ObsSpace API in `src/obsspace_mod.F90`; separate
  engines API in `src/engines/ioda/fortran/`) and Python bindings
  (`pyioda` in `src/engines/ioda/python/`, utilities in
  `src/python/pyiodautils/`).
- `src/IodaTrait.h` so apps that need only obs-side infrastructure
  (without UFO operators) can use IODA directly.

## How it fits into the bundle

- **Depends on:** `oops`, eckit, fckit, HDF5 (parallel), NetCDF, MPI;
  optionally odc (ODB engine) and the bufr library (BUFR engine).
- **Depended on by:** `ufo`, `iodaconv`, every model wrapper that runs
  DA against observations.
- **Build position:** after oops, before ufo. Test data comes from the
  `ioda-data` sidecar repo (bundle option `ENABLE_IODA_DATA`, default
  ON).

## Key directories

- `src/` — the IODA library.
  - Top level: `ObsSpace.{h,cc}`, `ObsVector.{h,cc}`,
    `ObsDataVector.h`, `ObsIterator.{h,cc}`, `ObsSpaceParameters.h`,
    `ObsDataIoParameters.{h,cc}`, `IodaTrait.h`; Fortran ObsSpace API
    (`obsspace_mod.F90`, `obsspace_interface.f`,
    `obsdatavector_int_mod.f90`).
  - `containers/` — OSDF data frame: `IFrame.h` (abstract slicing
    interface), `FrameCols`/`FrameRows` (+`*Data`), `ViewCols`/
    `ViewRows`, `ColumnMetadata`, `Datum`/`DataRow`.
  - `core/` — utilities: `IodaUtils`, `FileFormat`, `ObsDimInfo`,
    parameter traits, the C/Fortran glue for ObsSpace
    (`obsspace_f.{h,cc}`).
  - `distribution/` — `DistributionFactory`,
    `DistributionParametersBase.h`, and all distribution classes
    listed above, plus `DistributionUtils`.
  - `engines/` — the ioda-engines subpackage:
    - `engines/ioda/include/ioda/` — public headers: `Group.h`,
      `ObsGroup.h`, `Variables/`, `Attributes/`, `Types/`, and
      `Engines/` (backend headers + `ReaderFactory.h`,
      `WriterFactory.h`, `ReaderBase.h`, `WriterBase.h`,
      `EngineUtils.h`, `ReadH5File.h`, `WriteH5File.h`,
      `ReadOdbFile.h`, `WriteOdbFile.h`, `ReadBufrFile.h`,
      `ReadScriptFile.h`).
    - `engines/ioda/src/ioda/Engines/` — backend implementations:
      `HH/` (HDF5), `ObsStore/`, `ODC/`, `Bufr/`, `Script/`,
      `GenList.cpp`, `GenRandom.cpp`.
    - `engines/ioda/fortran/` — engines Fortran API
      (`ioda_engines_mod.f90`, group/variable/attribute modules).
    - `engines/ioda/python/` — pybind11 bindings building the
      `pyioda` package (`ioda`, `ioda_obs_space` modules).
    - `engines/ioda/src/mains/` — `upgrade/` (ioda file v2→v3
      upgrader) and `odc_converter/`.
    - `engines/Examples/` — Basic/Advanced usage examples (C++, C,
      Fortran, Python).
  - `reader/` — `ObsReader`, `OsdfFrameFacade`, and stages `load/`
    (`loadObsFromNetcdf`, `loadObsFromOdb`), `distribute/`
    (`distributeObs`), `filter/` (`filterObs`).
  - `writer/` — `ObsWriter` and stages `collect/` (`collectObs`),
    `save/` (`saveObs`, `saveObsToNetcdf`).
  - `ioPool/` — reader/writer pool framework (`ReaderPoolFactory`,
    `ReaderSinglePool`, `NonoverlappingReaderPool`,
    `WriterPoolFactory`, `WriterSinglePool`).
  - `obsIoPool/` — `ObsIoPool` glue between ObsSpace and the pools.
  - `mains/` — executables: `validator/` (IODA file validator),
    `filterObs/`, `odfDemo/`, `buildInputFileSet`,
    `randomObsVectorIO`, `timeIodaIO`.
  - `python/pyiodautils/` — pure-Python utilities (`file_merge.py`).
  - `example/` — minimal Fortran ObsSpace usage example.
- `share/ioda/yaml/` — ODB ctest YAMLs and the `odb_*_name_map.yaml`
  mapping files used by the ODC engine, plus `validation/`.
- `test/` — headers/mains for ctests, mirroring src: `ioda/` (ObsSpace
  tests), `containers/` (Osdf* tests), `distribution/`, `engines/`,
  `ioPool/`, `reader/`, `writer/`, `mains/`, `python/`, and
  `testinput/` YAMLs (~460+ ctests registered in
  `test/CMakeLists.txt`).
- `tools/` — `check_ioda_nc.py`, `ioda_compare.sh`,
  `ioda_compare_nc.py`, `ioda_compare_odc_with_netcdf.py`,
  `compare-odbs/`, `retrieval_upgrader.py`, lint helpers.
- `docs/` — Doxygen config only; real docs live in jedi-docs.

## Key entry points

- `src/ObsSpace.{h,cc}` — the primary user-facing class
  (`class ObsSpace : public oops::ObsSpaceBase`). Walking through
  `ObsSpace.cc` shows how files are opened, distributions applied,
  and variables loaded/saved.
- `src/ObsSpaceParameters.h` — the YAML schema for `obs space:`
  blocks (with `src/ObsDataIoParameters.h` for the `obsdatain:` /
  `obsdataout:` engine sub-blocks).
- `src/IodaTrait.h` — standalone Traits header for obs-only apps.
- `src/distribution/DistributionFactory.h` — distribution
  registration; `DistributionParametersBase.h` for the YAML schema.
- `src/engines/ioda/include/ioda/Engines/ReaderFactory.h` and
  `WriterFactory.h` — backend engine selection/registration.
- `src/mains/validator/validate.cpp` — the IODA file validator CLI.
- `src/engines/ioda/src/mains/upgrade/` — the v2→v3 file upgrader.
- `src/engines/Examples/` — minimum-viable engines usage examples.

## Common tasks

- **Run the IODA tests** — `ctest -L ioda` (the project sets
  `CMAKE_DIRECTORY_LABELS ioda`); see `test/CMakeLists.txt` for the
  full list (~460+ ctests).
- **Inspect an IODA file** — `h5dump`/`ncdump` work on IODA HDF5
  files; `tools/check_ioda_nc.py` validates conventions; the compiled
  validator main checks files against YAML specs in
  `share/ioda/yaml/validation/`.
- **Compare output files** — `tools/ioda_compare_nc.py` (NetCDF vs
  NetCDF), `tools/ioda_compare_odc_with_netcdf.py`, `tools/compare-odbs/`.
- **Choose a distribution** — set `obs space.distribution.name:` in
  YAML (`Halo`, `RoundRobin`, …). See `src/distribution/` for options.
- **Upgrade old (v2) IODA files** — the upgrader main under
  `src/engines/ioda/src/mains/upgrade/`.
- **Write a new engine** — derive from `ReaderBase`/`WriterBase` in
  `src/engines/ioda/include/ioda/Engines/` and register with
  `ReaderFactory`/`WriterFactory`.
- **Convert raw obs to IODA format** — that's `iodaconv`'s job; see
  `jedi-knowledge/iodaconv.md`.

## Gotchas

- **OSDF reader fill-value fix (2026-09, ioda#1867):** the missing
  `getNcVarDefaultFillValue` specialisation for `char` was added, plus a
  compile-time guard so other data types can no longer fall through to the
  `int` specialisation implicitly. This was what broke `ufo_superob` and
  `ufo_superob_PE4` under the OSDF reader. New coverage in
  `test/reader/ReaderFillValue.h` and
  `test/mains/reader/TestReaderFillValue.cc` (fixture from ioda-data#258).
- **`ObsSpace.h` and `ObsVector.h` slimmed (2026-09, ioda#1852, #1868):**
  both headers shed includes. Downstream code that relied on picking up
  transitive includes through them may now fail to compile — add the
  include you actually use.
- **Test-fixture container settings (2026-09, ioda#1861, #1866, #1859):**
  local test fixtures gained a global container-setting hook, paired
  container tests had their YAML differences cleaned up, and an
  `observers` node was inserted into the test YAMLs. Relevant if you
  maintain ioda ctests or the OSDF/classic paired tests.
- IODA files are HDF5 with specific group/variable conventions —
  arbitrary HDF5 files won't work. Use `iodaconv` outputs or another
  IODA-aware tool.
- Variable naming conventions matter: observed values, errors, QC
  flags, H(x), and metadata each live in conventional groups
  (`ObsValue`, `ObsError`, `EffectiveQC`, `hofx`, `MetaData`, …).
  Mismatched names produce silent zeros downstream.
- `Halo` distribution is the typical choice for high-resolution global
  models; `RoundRobin` is a fallback for testing. Picking the wrong
  one can dramatically change MPI load balance.
- datetime variables use epoch-style units (e.g.
  `seconds since 1970-01-01T00:00:00Z`); older files carrying bare
  ISO-string units need upgrading.
- **`ObsSpace::put_db` requires `dimList` (2026-05):** the API was
  tightened to require an explicit dimension list (ioda#1731). All ufo
  and ioda call sites were updated; any local branch calling `put_db`
  must add the `dimList` argument.
- **CDA: obs outside the shifted window are discarded (2026-05):**
  `src/ObsSpace.cc` removes observations outside the shifted
  assimilation window when running CDA (ioda#1722). Affects obs counts
  in CDA experiments.
- **`DistributionParametersBase` name Parameter has no default
  (2026-05):** you must explicitly set the `name` key in your YAML
  `distribution:` block (ioda#1742).
- **IFrame slicing interface (2026-06, ioda#1768):**
  `src/containers/IFrame.h` defines an abstract interface for slicing
  the data frame; `FrameCols` and `FrameRows` implement it. If you
  subclass either, add the `IFrame` virtual overrides.
- **ObsWriter added (2026-06, ioda#1755 area):**
  `src/writer/ObsWriter.{cpp,hpp}` and
  `test/mains/writer/TestObsWriter.cc` introduce a dedicated writer
  class; `ObsSpace::save` gained a flag to preserve the MPI
  distribution when writing (ioda#1755).
- **OSDF put_db no-op on empty vector (2026-06, ioda#1767):** `put_db`
  is a no-op when the data vector is empty, preventing spurious errors
  in OSDF-based flows that write optional fields.
- **Fortran API split (2026-06, ioda#1756):** the ObsSpace Fortran API
  (`src/obsspace_mod.F90`) and the ioda-engines Fortran API
  (`src/engines/ioda/fortran/`) are now two separate independent
  packages. If your Fortran code used the combined interface, update
  your `USE` statements to target the appropriate package.
- **OSDF supports Channel-only 1D variables (2026-06, ioda#1773):** the
  OSDF-based ObsSpace `put_db`/`get_db` paths in `src/ObsSpace.cc` now
  handle variables dimensioned solely by `Channel` (e.g. per-channel
  metadata): `get_db` returns `numChans` values instead of
  `numLocs * numChans`, and `put_db` broadcasts the per-channel value
  across the column. New `iodatest_time_io_osdf_mpi{1,2,4}` ctests cover
  it (reference data in ioda-data#245). Another step toward OSDF/classic
  ObsSpace parity.
- **ODB writes output Null instead of missing (2026-06, ioda#1733):**
  the ODB write path now emits Null for missing values; downstream ODB
  consumers comparing against old reference files will see diffs
  (matching reference updates landed in ioda-data#243).

## Further reading

- jedi-docs.jcsda.org → JEDI Components → IODA.
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/ufo.md`,
  `jedi-knowledge/iodaconv.md`, `jedi-knowledge/ioda-data.md`,
  `jedi-knowledge/jedi-bundle.md`.
