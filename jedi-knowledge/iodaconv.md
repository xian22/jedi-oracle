# iodaconv (ioda-converters)

**Repository:** https://github.com/jcsda-internal/ioda-converters
**Branch tracked by bundle:** develop
**Role in JEDI:** converters that turn raw observation files (BUFR, NetCDF, HDF, agency-specific formats) into IODA-format files consumable by JEDI. Optional bundle component.

## What it is

`ioda-converters` is the "import" half of the observation pipeline.
JEDI consumes observations through IODA in a standardized HDF5 layout;
this repo contains the scripts and tools that translate from upstream
operational/research formats (BUFR, NCEP PrepBUFR, GOES-R NetCDF,
sondes, etc.) into that IODA layout.

## How it fits into the bundle

- Built only when `BUILD_IODA_CONVERTERS=ON` (default `OFF`).
- Produces files consumed by `ioda` and `ufo` at runtime.
- See `jedi-knowledge/ioda.md` for the consumer side.

## Common tasks

- **Enable converters** — pass `-DBUILD_IODA_CONVERTERS=ON` to cmake,
  or let `/getCode` do it.
- **Convert a BUFR file to IODA format** — the typical workflow uses
  the per-instrument scripts under `src/`. See the in-repo README
  after cloning.

## Gotchas

- **OSW converter updated for the NOAA-L2 Muon product (2026-09,
  iodaconv#1810):** `src/hdf5/osw_2ioda.py`.
- Many converters depend on external libraries (NCEPbufr, eccodes)
  beyond the bundle's standard deps.

## Further reading

- `jedi-knowledge/ioda.md`, `jedi-knowledge/ufo.md`
