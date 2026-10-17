# pyiri-jedi

**Repository:** https://github.com/jcsda-internal/pyiri-jedi
**Branch tracked by bundle:** develop (cloned RECURSIVE)
**Role in JEDI:** Python-IRI (International Reference Ionosphere) coupling for ionospheric DA in JEDI. Optional bundle component.

## What it is

`pyiri-jedi` provides a JEDI-side wrapper around PyIRI, a Python
implementation of the International Reference Ionosphere model. It
enables ionospheric data assimilation experiments — using IRI as a
background and assimilating observations like GPS-TEC.

The bundle clones it `RECURSIVE`, so submodules come along.

## How it fits into the bundle

- Built only when `BUILD_PYIRI=ON` (default `OFF`).
- Adds an ionosphere model interface usable by oops/ufo for space-weather
  DA experiments.

## Common tasks

- **Enable PyIRI** — pass `-DBUILD_PYIRI=ON` to cmake, or let `/getCode`
  do it.

## Gotchas

- **Densities are now in SI units, m⁻³ (2026-10, pyiri-jedi#201):**
  background/state files store `dene`/`deni` in m⁻³ (previously cm⁻³)
  and carry a `units = 'm-3'` attribute. `FieldsPyiri::read` asserts the
  `deni` units attribute is `m-3`, and the Python GeoVaLs writer asserts
  the same, so **files made before this change fail to load** and must
  be regenerated. The GeoVaLs writer's `density_unit` default is now
  `m-3` (`cm-3` still available). The VTEC operator converts to cm⁻³
  internally before integrating, to avoid overflow. The LETKF test
  YAMLs were updated to match.
- **Skylab background/ensemble scripts (2026-09, pyiri-jedi#173):**
  scripts to build the background and ensemble inputs for Skylab
  experiments.
- **numpy 2.0 compatibility (2026-09, pyiri-jedi#199):** fixes in the
  Python layer for numpy 2.x. If you hit numpy API errors, make sure this
  commit is in your checkout.
- **Interpolator interface updated (2026-09, pyiri-jedi#185):**
  `src/pyiri-jedi/Model/InterpolatorPyiri.h` was adapted to oops#3333,
  which moved the interpolator caches out of `GeometryData`. See
  `jedi-knowledge/oops.md`.
- Recursive clone — submodules pull additional code; expect a longer
  clone.
- Python interpreter dependency — be sure your build environment's
  Python matches what JEDI's other Python bits use (see
  `Python3_FIND_STRATEGY LOCATION` in the bundle CMakeLists).

## Further reading

- `jedi-knowledge/oops.md`, `jedi-knowledge/ufo.md`
