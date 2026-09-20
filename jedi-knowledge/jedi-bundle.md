# jedi-bundle

**Repository:** https://github.com/jcsda-internal/jedi-bundle
**Branch tracked by bundle:** n/a — top-level repo (clone `develop`)
**Role in JEDI:** the umbrella build that pulls every JEDI repo together and orchestrates the cmake/ecbuild build of the full stack.

## What it is

The "bundle" is not a code library — it's a thin wrapper around `ecbuild`'s
bundle mechanism. The tracked content is tiny: `CMakeLists.txt`, `README.md`,
`LICENSE`, `.gitignore`, and one CI workflow
(`.github/workflows/start-jedi-ci.yaml`). Everything else in the directory is
a gitignored sub-repo clone or build artifact.

`CMakeLists.txt` (project version 8.0.0, requires CMake ≥ 3.14 and
ecbuild ≥ 3.6) declares each JEDI repository (URL, branch/tag, optional
flags); `ecbuild_bundle_initialize()` / `ecbuild_bundle_finalize()`
orchestrate cloning and building them in dependency order. When you "build
JEDI", you actually build the bundle. Defaults set in the file:
`ECBUILD_DEFAULT_BUILD_TYPE Release`, `ENABLE_MPI ON`, `ENABLE_OMP ON`,
`Python3_FIND_STRATEGY LOCATION`. It includes
`$ENV{jedi_cmake_ROOT}/share/jedicmake/Functions/git_functions.cmake`, so an
external **jedi-cmake** installation (normally from spack-stack) must be in
the environment before configuring.

## How it fits into the bundle

Every other JEDI core repo in `jedi-knowledge/` is a sub-repository of the
bundle. Authoritative `ecbuild_bundle( PROJECT … )` list as of 2026-09 (all
`jcsda-internal`, `BRANCH develop UPDATE`, unless noted):

**Always built (no option flag), in declaration order:**

| Project | Source | Notes |
|---|---|---|
| gsw | jcsda-internal/GSW-Fortran | |
| oops | jcsda-internal/oops | |
| jedi-model-data | jcsda-internal/jedi-model-data | shared model test data |
| vader | jcsda-internal/vader | |
| saber | jcsda-internal/saber | |
| crtm | **jcsda/CRTMv3** (public org) | |
| ioda-data | jcsda-internal/ioda-data | |
| ioda | jcsda-internal/ioda | |
| ufo-data | jcsda-internal/ufo-data | |
| ufo | jcsda-internal/ufo | |

**Optional, default ON** (so a default configure builds all of these):

- `BUILD_FV3` → `fv3-jedi-lm` (jcsda-internal/fv3-jedi-linearmodel),
  `fv3-jedi-data`, `fv3-jedi`.
- `BUILD_SOCA` → `soca` (`RECURSIVE` — submodules).
- `BUILD_MPAS` → `MPAS` (MPAS-Dev/MPAS-Model, **TAG v8.4.2**),
  `mpas-jedi-data`, `mpas-jedi`. Also sets `MPAS_DOUBLE_PRECISION ON`,
  `MPAS_CORES "init_atmosphere atmosphere"`, `MPAS_OPENMP ON`.
- `BUILD_PYIRI` → `pyiri-jedi` (develop, `RECURSIVE`).
- **`coupling`** has no flag of its own — it is cloned only when
  `BUILD_FV3 AND BUILD_SOCA` are both ON.
- `ENABLE_IODA_DATA` (ON), `ENABLE_UFO_DATA` (ON) — take test data from the
  `ioda-data` / `ufo-data` repos instead of a tarball. The data repos
  themselves are cloned unconditionally; these flags only control how the
  test data is consumed. (There is **no** `ENABLE_FV3_JEDI_DATA` — the
  fv3-jedi-data clone is governed by `BUILD_FV3`.)

**Optional, default OFF:**

- `BUILD_GSIBEC` → `gsibec` (geos-esm/GSIbec, TAG 1.4.4)
- `BUILD_RTTOV` → `rttov` (jcsda-internal/rttov, develop)
- `BUILD_OASIM` → `oasim` (jcsda-internal/oasim, develop)
- `BUILD_ROPP` → `ropp-ufo` (jcsda-internal/**ropp-test**, develop)
- `BUILD_MIST` → `mist` (jcsda-internal/mist, develop)
- `BUILD_IODA_CONVERTERS` → `iodaconv` (jcsda-internal/ioda-converters, develop)

eckit / fckit / atlas entries exist but are **commented out** — they come
from the environment (spack-stack), not the bundle.

## Key directories

- `CMakeLists.txt` — the manifest (the only substantive tracked file). The
  oracle reads this to know which repos exist, where to clone them from, and
  which branch/tag to use.
- `.github/workflows/start-jedi-ci.yaml` — nightly CI (cron 01:30
  America/Denver, plus manual dispatch); launches `JCSDA-internal/jedi-ci`
  with an INTEGRATION test strategy.
- `<sub-repo>/` dirs (oops/, ufo/, ioda/, …) — gitignored clones; see each
  repo's own brief.
- `test-data-release/` — gitignored local directory holding released
  CRTM coefficient/fix data (e.g. `fix_REL-3.1.2.0/`); can be very large
  (tens of GB). Present only if the data has been fetched.
- There is **no** `cmake/` directory in the repo, despite `CMakeLists.txt`
  appending one to `CMAKE_MODULE_PATH` — helper functions come from the
  external jedi-cmake package.

## Key entry points

- `CMakeLists.txt` — read top-to-bottom for the full repo/flag picture.
- Configure with `ecbuild <flags> /path/to/jedi-bundle` from a separate
  build dir, after loading a jedi/spack-stack environment.

## Common tasks

- **What repos does the bundle include?** — read `CMakeLists.txt` and list
  every `ecbuild_bundle( PROJECT … )` line (or see the table above).
  `/jedi-getCode` does this for you.
- **Enable an optional package** — pass `-DBUILD_<NAME>=ON` to ecbuild/cmake,
  or flip the `option(...)` default in `CMakeLists.txt`. `/jedi-getCode` can
  do this when cloning.
- **Update branch/tag for a sub-repo** — edit the `BRANCH` or `TAG` argument
  on its `ecbuild_bundle(...)` line, then re-clone or `git checkout` inside
  the sub-repo dir. The `UPDATE` keyword makes ecbuild pull the branch on
  each configure.
- **Build the whole thing** — see `jedi-knowledge/jedi-tips.md`,
  `jedi-tools/buildscripts/` (setup.sh + build_jedi_skylab.sh), and
  https://jedi-docs.jcsda.org/ for canonical instructions (depends on
  site/module environment).

## Gotchas

- **MPI is probed once at bundle scope (2026-09, jedi-bundle#159):**
  the top-level `CMakeLists.txt` now calls
  `find_package( MPI COMPONENTS C CXX Fortran )` right after
  `ecbuild_bundle_initialize()`, so every sub-repo's own
  `find_package(MPI)` still configures its target but skips the
  `try_compile` probes. A configure-time speedup; if a sub-repo picks up
  the wrong MPI, look at the bundle-level call first.
- `mpas` pins **TAG v8.4.2** (bumped 2026-09, PR #160; previously
  v8.4.0 from PR #145). If a local
  `jedi-bundle/MPAS` checkout is older (note the uppercase directory
  name — the ecbuild PROJECT is `MPAS`), `git fetch && git checkout v8.4.2`
  before rebuilding.
- `BUILD_MPAS` was briefly made default-OFF (PR #136) but is **ON** in the
  current `CMakeLists.txt` — always check the file, not memory.
- Optional packages rttov and oasim require licensed source / private repo
  access; cloning fails without it.
- The ROPP project name is `ropp-ufo` but the source repo is `ropp-test`,
  and the flag is `BUILD_ROPP` — three different names for one package.
- `UPDATE` on a bundle entry means every fresh configure does a `git pull`
  in that sub-repo — local uncommitted work on a tracked branch can be
  disturbed; work on feature branches.
- FEMPS and the GFDL atmos cubed-sphere fork were removed from the bundle
  (2025); briefs or docs that mention them as bundle members are stale.

## Further reading

- https://jedi-docs.jcsda.org/ — official JEDI docs (cloned locally in
  `jedi-docs/` if present)
- `jedi-knowledge/<repo>.md` for any sub-repo
- `jedi-knowledge/jedi-tools.md` — buildscripts that drive bundle builds
