# ufo-data

**Repository:** https://github.com/jcsda-internal/ufo-data
**Branch tracked by bundle:** develop
**Role in JEDI:** the test-data sidecar for `ufo` — IODA-format observations and matching geoVaLs / obs-diagnostics used by UFO's instrument and unit tests.

## What it is

A pure data repo (project version 1.13.0). The CMakeLists declares an
`ecbuild_install_project`; content lives under `testinput_tier_1/`. The
bundle defines `ENABLE_UFO_DATA` (default ON) which controls whether UFO
tests take their data from this repo or from a downloaded tarball — note
the bundle clones the repo unconditionally either way.

This is the largest of the data sidecars — UFO's test matrix covers many
instruments and operator variants, so the fixture set is
correspondingly large (~690 files at last scan).

Also at top level: `CI/` (CodeBuild buildspec, `clone.sh`, CDash glue for
this repo's own CI) and `update_part2.py`, a small netCDF helper used when
bulk-renaming variables in data files.

## How it fits into the bundle

- **Consumed by:** `ufo` (its `test/` suite reads files under
  `testinput_tier_1/`).
- **Build position:** before `ufo` tests run.

## Key directories

- `testinput_tier_1/` — flat directory, split by content category and
  instrument prefix:
  - `*_obs_*` — observation values.
  - `*_geoval(s)_*` — geovals (model values interpolated to obs
    locations).
  - `*_obsdiag_*` — observation diagnostics.
  - bias-correction (`satbias_*`, `aircraftBCOffline_*`) and covariance
    (`Rcov_*`) files.
  - Per-instrument grouping (sondes, aircraft, AMSU-A/B, ATMS, IASI,
    CrIS, AIRS, ABI, AHI, GMI, AMSR2, SSMIS, MHS, GNSS-RO, AOD, SMAP,
    SST, scatwind, etc.).

## Common tasks

- **Use the tarball instead of the repo** — pass `-DENABLE_UFO_DATA=OFF`
  to the bundle; UFO tests then use the tarball fallback path. (The repo
  is still cloned by the bundle's `ecbuild_bundle` line unless you
  comment it out.)
- **Update fixtures** — modify under `testinput_tier_1/`, bump the
  project version in `CMakeLists.txt`. UFO PRs that change reference
  values usually need a paired ufo-data PR.

## Gotchas

- This is a large repo — full clone takes time and disk.
- File naming conventions are referenced by literal strings in UFO's
  test YAMLs; renaming silently breaks tests.
- **Large GNSSRO files deleted (2026-05, ufo-data#562):** oversized
  GNSSRO geoval files were removed in favor of smaller variants; old UFO
  branches whose test YAMLs reference the deleted names will fail.
- **Met Office surface-cloud fixtures rewritten (2026-09, ufo-data#596):**
  `met_office_surfacecloud_cloud_base_height_check.nc4` and
  `met_office_surfacecloud_cloud_column_check.nc4` had their 1-D variables
  removed where they collided with a same-named 2-D variable. A UFO branch
  that still reads the 1-D form of those variables will fail to find them.
- Recent additions track new UFO obsfunctions (`LinearTimeInterpolate`,
  `Statistic`, `TimeBinner`, `CircularDifference`, vertical smoothing) —
  if a brand-new UFO test can't find its data, pull this repo too.

## Further reading

- Related briefs: `jedi-knowledge/ufo.md`, `jedi-knowledge/ioda-data.md`,
  `jedi-knowledge/jedi-bundle.md`.
