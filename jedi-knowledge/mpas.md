# mpas (MPAS-Model)

**Repository:** https://github.com/MPAS-Dev/MPAS-Model
**Pin tracked by bundle:** tag `v8.4.2` (pinned tag, not a branch)
**Role in JEDI:** the MPAS atmospheric forecast model itself — the upstream NCAR/LANL MPAS-Atmosphere code that `mpas-jedi` wraps for JEDI integration.

## What it is

MPAS (Model for Prediction Across Scales) is the unstructured-mesh
weather and climate model maintained by NCAR and LANL. The bundle pulls
its `atmosphere` core (and `init_atmosphere` for preprocessing) so that
`mpas-jedi` has a real forecast model to wrap. Other MPAS cores
(`core_ocean`, `core_seaice`, `core_landice`, `core_sw`, `core_test`)
ship in the upstream repo but the bundle does not build them — see the
`MPAS_CORES` setting in jedi-bundle's CMakeLists
(`init_atmosphere atmosphere`).

## How it fits into the bundle

- **Built when:** `BUILD_MPAS=ON` (default ON).
- **Built with:** `MPAS_DOUBLE_PRECISION=ON`, `MPAS_OPENMP=ON`,
  `MPAS_CORES="init_atmosphere atmosphere"` (defaults set in
  `jedi-bundle/CMakeLists.txt`; MPAS's own CMake defaults differ —
  `MPAS_CORES=atmosphere`, `MPAS_OPENMP=OFF`).
- **Depended on by:** `mpas-jedi`. The bundle builds MPAS first, then
  `mpas-jedi-data`, then `mpas-jedi`.

## Key directories

- `CMakeLists.txt` — entry point for the cmake build path the bundle
  uses. Notable options: `MPAS_CORES`, `MPAS_DOUBLE_PRECISION` (default
  TRUE), `MPAS_OPENMP` (default OFF), `DO_PHYSICS` (default TRUE),
  `MPAS_USE_PIO` (default OFF), `MPAS_PROFILE` (GPTL, default OFF).
- `src/core_atmosphere/` — atmospheric solver (the bundled core):
  `dynamics/`, `physics/` (as `chemistry/`, `diagnostics/`, etc.),
  `mpas_atm_core.F`, `Registry.xml`.
- `src/core_init_atmosphere/` — initial-condition / preprocessing
  core (also built when bundle sets `MPAS_CORES`).
- `src/core_ocean/`, `src/core_seaice/`, `src/core_landice/`,
  `src/core_sw/`, `src/core_test/` — other cores; **not** built by
  the bundle.
- `src/framework/` — shared infrastructure (data types, I/O streams,
  communication); `src/driver/`, `src/operators/`, `src/tools/`,
  `src/external/` — supporting code.
- `Registry.xml` (one per core, e.g.
  `src/core_atmosphere/Registry.xml`) — declarative variable /
  namelist / stream config that drives MPAS code generation.

## Key entry points

- `src/core_atmosphere/mpas_atm_core.F` and
  `mpas_atm_core_interface.F` — atmosphere core init/run/finalize, the
  surface `mpas-jedi` talks to.
- `src/core_atmosphere/Registry.xml` — where model variables and
  namelist options are declared; first place to look when a field or
  config option "doesn't exist".

## Common tasks

- **Build via the bundle** — automatic when `BUILD_MPAS=ON`; see the
  flags listed above.
- **Add another MPAS core to the bundle build** — set
  `MPAS_CORES="init_atmosphere atmosphere ocean"` in
  `jedi-bundle/CMakeLists.txt` (and ensure required deps are
  available).

## Gotchas

- **Pinned to a tag, not a branch.** The bundle pins `TAG v8.4.2`
  (bumped from v8.4.0 in 2026-09, jedi-bundle#160)
  (the official release containing the MPAS-model#1420 fix that an
  earlier commit pin was waiting for). Implications:
  - `/jedi-updateRepos` skips this repo — there is no `develop` to
    pull; the checkout sits at the tag.
  - To move to a newer release, edit the `TAG` field in
    `jedi-bundle/CMakeLists.txt`.
  - Don't switch the cloned `mpas/` to another branch and expect the
    bundle (and `mpas-jedi`, which requires `MPAS 8.0` via
    `find_package`) to still work.
- The MPAS build needs a working parallel NetCDF I/O stack (PIO is
  optional via `MPAS_USE_PIO`, default OFF in the cmake path). Module
  setup at HPC sites usually handles this; on local machines it's a
  frequent build blocker.
- `mpas-jedi`'s test data was generated with a **single-precision**
  MPAS build (see the mpas-jedi brief) while the bundle defaults to
  double precision — keep precision consistent between MPAS, mpas-jedi,
  and the reference outputs you compare against.

## Further reading

- https://mpas-dev.github.io/ — upstream docs.
- Related briefs: `jedi-knowledge/mpas-jedi.md`,
  `jedi-knowledge/mpas-jedi-data.md`, `jedi-knowledge/jedi-bundle.md`.
