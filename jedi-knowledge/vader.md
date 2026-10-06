# vader

**Repository:** https://github.com/jcsda-internal/vader
**Branch tracked by bundle:** develop
**Role in JEDI:** Generic, model-independent variable-derivation library. Converts between meteorological/oceanographic variables (e.g. specific humidity ↔ relative humidity, exner ↔ pressure, T ↔ θ) so models, observation operators, and B-matrix code don't each reinvent the same transforms.

## What it is

VADER (VAriable DErivation Repository) provides reusable variable transforms
("recipes") that operate on Atlas FieldSets. Each recipe declares an
output variable name plus required input variable names, and its
`executeNL` / `executeTL` / `executeAD` methods do the math (and its
TL/AD where needed). Vader assembles a `cookbook` (output variable →
ordered list of candidate recipes) and, given an input FieldSet and a
list of desired output variables, plans and executes the necessary chain
of recipes.

The intent is for every JEDI model interface to delegate variable changes
to VADER instead of carrying private copies of "compute air_temperature
from potential_temperature and exner". Models can override / supplement
the default cookbook in YAML.

## How it fits into the bundle

- **Depends on:** `oops 1.10.0`, atlas (via oops), MPI, NetCDF, Boost.
  Optionally `gsw` (the GSW-Fortran toolbox) — when present, marine
  recipes (TEOS-10 / ocean conversions) get compiled in. The vader
  project version itself is 1.7.0 (what saber's `find_package` pins).
- **Depended on by:** `saber`
  (`find_package(vader 1.7.0 REQUIRED)` in `saber/CMakeLists.txt`), and
  most model interfaces (fv3-jedi, mpas-jedi, soca) use VADER inside
  their `VariableChange` / `LinearVariableChange` implementations.
- **Build position:** between oops and saber/models.

## Key directories

- `src/vader/` — core C++ library (the `vader::` namespace).
  - `vader.h`, `vader.cc` — the `Vader` class: holds a cookbook, plans
    recipe chains, executes them on an Atlas FieldSet.
  - `RecipeBase.h`, `RecipeBase.cc` — abstract recipe interface; every
    recipe derives from this.
  - `DefaultCookbook.h` — the default mapping
    `oops::Variable → [RecipeName, ...]`. Glance at this to see what is
    offered out-of-the-box.
  - `VaderParameters.h` — YAML configuration schema.
  - `recipes/` — atmosphere/microphysics recipes (~114 files):
    `AirTemperature_*.cc`, `AirPotentialTemperature_*.cc`,
    `AirPressure*`, `Cloud*MixingRatio*`, `WaterVapor*`,
    `Geopotential*`, `HydrostaticExnerLevels`, `EastwardWindAt10m` /
    `NorthwardWindAt10m` / `WindReductionFactorAt10m`,
    `ParticulateMatter2p5`, etc. Multiple variants per output variable
    (suffix `_A`, `_B`, `_C`) take different input combinations.
  - `betaNames_CRTMRecipes/` — a second recipe directory (~77 files) for
    the variables CRTM consumes (hydrometeor layer mass contents,
    effective radii, surface fractions, surface winds, …) and the marine
    temperature recipes `SeaWaterTemperature` /
    `SeaWaterPotentialTemperature` (gsw-gated). Check here as well as
    `recipes/` before writing a new recipe.
- `src/mo/` — Met Office–contributed atmospheric recipes and helper code
  (`eval_air_density`, `eval_air_temperature`, `eval_exner`,
  `eval_hydrostatic_balance`, `eval_moisture_incrementing_operator`,
  `eval_sat_vapour_pressure`, lookup tables `svp_lookup.h`,
  `dlsvp_lookup.h`, etc.). `src/mo/recipes/` contains aerosol-related
  dust recipes.
- `src/OceanConversions/` — Fortran↔C++ glue for marine recipes.
  `OceanConversions.interface.F90` / `.h` wrap GSW-Fortran calls; only
  built when `gsw` is found.
- `test/vader/` — unit tests: `TestVader.cc`, `TestRecipe.cc`,
  `TestPlanVariable.cc`, `TestPrintVader.cc`, `TestUtils.cc`, plus an
  `mo/` subdir (`TestCheckLookupTables.cc` for the MO lookup tables).
- `test/testinput/` — YAML configuration fixtures used by the tests.
- `tools/cpplint.py` — coding-norms checker.

## Key entry points

- `src/vader/vader.h` — the `vader::Vader` class. Its
  `changeVar(atlas::FieldSet &, oops::Variables &)` method (returns the
  variables it could not produce) is the main NL API model code calls;
  `changeVarTraj` / `changeVarTL` / `changeVarAD` are the trajectory
  and TL/AD counterparts.
- `src/vader/DefaultCookbook.h` — the canonical list of "what variables
  can VADER produce by default and via which recipe(s)".
- `src/vader/RecipeBase.h` — the recipe contract; copy this when
  authoring a new recipe.
- `src/vader/recipes/AirTemperature.h` + `AirTemperature_A.cc` /
  `_B.cc` / `_C.cc` — clean, simple example showing the multi-variant
  pattern.
- `src/mo/eval_air_pressure.cc` / `.h` — example of a Met Office–style
  eval helper (used by recipes).
- `test/vader/TestVader.cc` — shows how a client constructs a `Vader`
  and exercises `changeVar`.

## Common tasks

- **Run vader tests** — they are gated by
  `find_package(jedi-model-data 1.0.0)`. With the bundle's
  `jedi-model-data` cloned, run `ctest -R vader_` in the build tree.
- **Add a new recipe**:
  1. Add `recipes/MyVariable.h` + `recipes/MyVariable_A.cc` (TL/AD
     methods if linearizable).
  2. List the sources in `src/CMakeLists.txt` (there is no per-subdir
     CMakeLists under `src/vader/`).
  3. Add an entry to `DefaultCookbook.h` mapping the output
     `oops::Variable` to the new recipe name.
  4. Add a test recipe entry under `test/`.
- **Override the cookbook from YAML** — pass a `cookbook` key in the
  `vader` config block; see comments in `src/vader/vader.h`
  (`configCookbookKey`, `configModelVarsKey`).
- **Hand model-specific constants to recipes** — pass them under
  `model data` in YAML; recipes look them up at execution.
- **Run cpplint** — `ctest -R vader_coding_norms` (uses
  `tools/cpplint.py`).

## Gotchas

- **Marine temperature linear variable change (2026-10, vader#208):**
  `betaNames_CRTMRecipes/SeaWaterTemperature_B.cc` adds TL/AD for marine
  in-situ ↔ potential temperature, and the marine temperature recipes
  moved from `recipes/` into `betaNames_CRTMRecipes/`. Test data in
  jedi-model-data#12.
- VADER works on Atlas `FieldSet`s using `oops::Variable` names. The
  variable name string must match what other JEDI components expect —
  variable-name drift is a frequent debugging cause.
- Marine recipes silently disappear if `gsw` is not found at configure
  (status line: `GSW NOT FOUND: Excluding Marine Recipes`). If your test
  expects ocean conversions, verify GSW is in the build.
- Recipes have ordered alternatives (`_A`, `_B`, ...). Vader picks the
  first whose ingredients are available — so adding inputs to a
  FieldSet can change which recipe fires. Use the `print` / planning
  APIs (see `TestPrintVader.cc`) to debug recipe selection.
- Tests are skipped (with status message) when `jedi-model-data` is
  missing — silently passing builds aren't actually testing anything.
- **`AirPotentialTemperature_B` formula bug fixed (2026-05):**
  `src/vader/recipes/AirPotentialTemperature_B.cc` — the formula was
  missing the `p0^kappa` factor (vader#365). If your experiment uses
  the `_B` variant of this recipe (i.e. the inputs resolve to it rather
  than `_A` or `_C`), check that your result is numerically consistent
  with the fixed formula before comparing against old references.
- **Virtual temperature formula fixed (2026-06, vader#362):**
  `src/vader/recipes/AirVirtualTemperature_A.cc` — the formula was
  incorrect. This was a real bug fix; any workflow that uses the
  `AirVirtualTemperature_A` recipe should expect changed output values
  and updated reference outputs (e.g. the fv3-jedi convertstate refs).
- **Level ordering for wind-at-10m recipes (2026-06, vader#370):**
  `EastwardWindAt10m.h/.cc`, `NorthwardWindAt10m.h/.cc`, and
  `WindReductionFactorAt10m_A.cc` now handle the "are levels top-down?"
  question explicitly. New test inputs added:
  `test/testinput/recipe_uwind_at_10m_A_topdown.yaml` and
  `recipe_vwind_at_10m_A_topdown.yaml`. If your model passes top-down
  level ordering, the recipe now correctly interprets it (uses the
  corresponding fixture `wind_at_10m_A_flipped.nc` from `jedi-model-data`).

## Further reading

- jedi-docs.jcsda.org: VADER section under "JEDI Components".
- Related briefs: `jedi-knowledge/oops.md`, `jedi-knowledge/saber.md`,
  `jedi-knowledge/gsw.md`, `jedi-knowledge/fv3-jedi.md`,
  `jedi-knowledge/mpas-jedi.md`, `jedi-knowledge/soca.md`,
  `jedi-knowledge/jedi-model-data.md`.
