# spack-stack

**Repository:** https://github.com/JCSDA/spack-stack
**Docs:** https://spack-stack.readthedocs.io/ (use the version switcher to
match the release you are building — commands drift between releases)
**Role in JEDI:** provides every compiled and Python dependency JEDI needs
(eckit/atlas/fckit, NetCDF/HDF5, MPI, ecflow, Python stack, …) as
environment modules. JEDI itself is built *on top of* a spack-stack
environment; jedi-bundle does not pin compilers or library versions.

For the step-by-step procedure on an unsupported machine, use the
**`/jedi-spackStack`** skill. This brief explains *how spack-stack works*
so build problems can be diagnosed.

## Which release goes with which JEDI

- jedi-bundle has no spack-stack version in it. The best current signal is
  **jedi-tools**: `buildscripts/setup/<host>_setup_<compiler>.sh` show the
  spack-stack release and compiler version each supported HPC uses today
  (see `jedi-knowledge/jedi-tools.md`).
- jedi-docs has a release table in
  `docs/using/building_and_running/building_jedi.rst` (anchor
  `spack_stack_jedi_release_table`), but it lags behind; treat it as a hint.
- `develop` JEDI → the newest spack-stack release. Tagged Skylab releases →
  the paired spack-stack release.
- spack-stack release branches are `release/<major>.<minor>`; patch tags
  (e.g. `2.1.1`) sit on those branches. List them with
  `git ls-remote --heads --tags https://github.com/JCSDA/spack-stack.git`.

## Repository layout (2.x)

- `setup.sh` — source this first. Sets `SPACK_STACK_DIR`, sets
  `SPACK_DISABLE_LOCAL_CONFIG=true` (ignore `~/.spack`), sets a cache path
  inside the checkout, and sources spack's `setup-env.sh`.
- `spack/` — spack itself (git submodule; spack-stack 2.1 carries a
  spack 1.x dev snapshot). `repos/builtin/` — spack's builtin package repo
  (submodule). Clone with `--recurse-submodules`.
- `repos/spack_stack/` — spack-stack's own packages, including the
  `*-env` bundle packages (`ewok-env`, `jedi-fv3-env`, `soca-env`, …).
- `spack-ext/` — the `spack stack …` command extension.
- `configs/templates/<name>/spack.yaml` — the root specs for an
  environment. Common ones: `skylab-dev` (JEDI + Skylab), `unified-dev`
  (everything: JEDI, UFS, GEOS, GSI, MET, …; the largest build).
- `configs/common/` — package requirements shared by every site
  (`packages.yaml`, plus `packages_gcc.yaml` / `packages_oneapi.yaml` /
  `packages_clang.yaml` selected by `--compiler`).
- `configs/sites/tier1/` — supported HPCs. `configs/sites/tier2/` —
  community-maintained sites, including the generic `linux.default` and
  `macos.default`, and OS-specific examples (e.g. an AWS Ubuntu site whose
  README lists apt prerequisites).
- `.github/workflows/*-ci-*.yaml` — **the closest thing to a tested
  recipe for that release.** The CI workflow's `spack external find`
  exclusions, `spack config add` lines (MPI provider, variants) and
  verification steps are more current than any docs page. Caveats: the
  workflows trigger on pull requests (mostly into `develop`), so a release
  branch is not re-tested after it is cut, and CI installs through a
  binary build cache, so packages it reuses are not rebuilt from source
  — a fresh build can hit failures CI never sees.
- `util/show_duplicate_packages.py` — finds unintended duplicate specs in a
  concretize log.

## Environment layout

`spack stack create env --site <site> --template <t> --name <n> --compiler <c>`
creates `envs/<n>/` containing:

- `spack.yaml` — the template's specs plus `include:` of the layers below.
  `spack config add …` with the env active writes here.
- `common/` — copy of `configs/common` (with the compiler-specific file).
- `site/` — copy of the site config. `spack external find --scope system`
  and `spack compiler find --scope system` write into `site/packages.yaml`
  **only while** `SPACK_SYSTEM_CONFIG_PATH=<env>/site` is exported. Unset
  it afterwards.
- After install: `install/` (packages) and the generated module tree. In
  2.x the module root used by setup scripts is `envs/<n>/modules/Core`
  (older 1.x docs say `install/modulefiles/Core`).

Environment names on supported hosts follow `ue-<compiler>-<version>`
(unified) / `se-…` (skylab) / `ce-…` / `ne-…`.

## Config layering and keywords

Precedence (highest first): the env's `spack.yaml` → `site/` → `common/`
→ spack defaults. Within `packages:`:

- `require:` — **hard**. The solver must satisfy it or fail. `one_of` /
  `any_of` give alternatives. `require::` (double colon) replaces lower
  layers instead of merging.
- `prefer:` — soft preference.
- `variants:` — soft preference for variant values. **Not a guarantee.**
- `buildable: false` — never build; must use an external.
- `externals:` — system installs spack may use instead of building.

`concretizer: unify: when_possible` (used by the big templates) lets
different root specs resolve to different versions of a dependency when
they cannot agree. Deliberate duplicates (several `esmf`, `crtm`, `fms`
builds in `unified-dev`) are expected.

## How the solver chooses (and why variants silently disappear)

Spack's concretizer minimizes a ranked list of criteria
(`spack/lib/spack/spack/solver/concretize.lp`, `opt_criterion`). For
packages being built, **"deprecated versions used" ranks above "default
values of variants not being used"**. Consequence: if keeping a default-on
variant forces a dependency onto a version upstream spack marks
`deprecated=True`, the solver will turn the variant **off** rather than use
the deprecated version — with no error, only a missing feature.

Diagnosing it:

1. Compare the concretized spec with the package defaults
   (`spack info <pkg>`, or `variant(...)` in its `package.py`).
2. `spack spec --fresh <pkg>+<variant>` — if it concretizes alone, the
   variant is satisfiable; look for a `deprecated version` warning.
3. Chase *why* the deprecated version is forced:
   `spack spec --fresh '<dep>@<newer>'` prints the chain of conflicting
   requirements.
4. Fix by turning the soft preference into a hard one
   (`packages:<pkg>:require:['+<variant>']`), accepting the deprecated
   version, or relaxing the constraint that forced it.

## Compilers

- `spack compiler find` registers **every** compiler it sees, including
  versions installed only as dependencies of the OS's default `gcc`
  package. Remove unwanted entries from `site/packages.yaml` (2.x stores
  compilers as `gcc:` externals there), or the solver may pick them.
- `--compiler gcc` on `create env` selects `common/packages_gcc.yaml`,
  which sets `prefer: gcc` for c/cxx/fortran.
- `spack stack check-preferred-compiler` after concretize confirms every
  compiled package uses the intended compiler.
- Pick the compiler *version family* the target release is used with on
  supported hosts (jedi-tools setup scripts), not the newest the OS ships.
  Brand-new compiler majors are untested and break older packages.
- macOS uses `--compiler clang`, which selects `common/packages_clang.yaml`.
  In 2.1 that file prefers `llvm` for C, C++ **and Fortran** (LLVM flang);
  older 1.x docs describe Apple clang paired with a Homebrew gfortran.
  Check `packages_clang.yaml` and the docs for the release you are
  building. jedi-docs notes the ewok graphics dependencies do not build
  with gcc on macOS. spack-stack 2.1 has no macOS CI workflow, so macOS
  builds of it are docs-guided only.

## Externals: when to use the system package and when not to

- spack-stack's CI excludes some system packages from `external find`
  (the exact list varies by release — read the CI workflow). Typical
  exclusions: `openssl`, `curl`, `python`, `bison`, `meson`, sometimes
  `cmake`. Spack builds those itself because system versions cause link or
  ABI problems.
- On an OS newer than the release was tested on, system tools
  (binutils, m4, autotools, libtool, gettext, perl) may be newer than
  packages expect. If a build fails inside one of them, delete that
  external from `site/packages.yaml` and re-concretize — spack will build
  its own known-good version.
- `mysql` only matters for running R2D2 locally (`ewok-env +mysql`).
- `texlive` only matters for building documentation.

## Install, rerun, modules

- `ulimit -S -s unlimited` in the build shell first.
- `spack install` is resumable: rerunning skips finished packages. Network
  timeouts during source download look like build failures — rerun.
- Run long installs under `screen` or `tmux`.
- **Judging success:** grepping the log for `Error` gives false positives
  (compiler warnings that quote error strings, Python byte-compile
  `SyntaxError`s for Python-2 helper scripts some packages ship). Look
  for `==> Error:` lines and "Skipping build of" lines. To be sure, check
  in `spack python` that every non-external spec in the environment
  reports `installed`.
- Afterwards: `spack module tcl refresh -y` (or `lmod`), then
  `spack stack setup-meta-modules`. These read spack's install database
  and write files; they can run from any shell with `setup.sh` sourced
  and the env activated, not only the one that ran the install.
- Loading: `module use <env>/modules/Core`, then load
  `stack-<compiler>/<version>`, then `stack-<mpi>/<version>`, then env
  modules (e.g. `jedi-fv3-env`, `ewok-env`), **each in a separate
  `module load` command**. The tree is hierarchical: a meta-module only
  adds the next level to `MODULEPATH` once it is loaded, so naming
  `stack-<mpi>` in the same command as `stack-<compiler>` fails with
  "Unable to locate a modulefile". 2.1 has no `stack-python`
  meta-module. Python comes in through the env modules.

## Known issues by release

Entries here are release-specific and will go stale. Record the date
observed and remove them when a newer release fixes them.

### 2.1 (observed 2026-10)

- **`ewok-env` concretizes without ecflow** with `unified-dev` on
  `linux.default`. Chain: `configs/common/packages.yaml` requires
  `cairo +pic` → only old autotools cairo (`@:1.17.4`) has that variant →
  harfbuzz's meson build needs newer cairo → only harfbuzz 2.9.1
  (deprecated) remains → `ecflow +ui` needs qt@5 which needs harfbuzz →
  solver drops ecflow (see "How the solver chooses"). Fix:
  `spack config add "packages:ewok-env:require:['+ecflow']"` (add `+mysql`
  in the same string if wanted), re-concretize, confirm in `spack.lock`.
- **OS newer than tested (e.g. an Ubuntu release after 24.04):** the
  default system gcc is a newer major than any supported host uses with
  2.1 (they use gcc 12–13). Install the distro's versioned `gcc-13`
  `g++-13` `gfortran-13` packages, make them the default with
  `update-alternatives`, and delete the newer gcc from
  `site/packages.yaml` after `spack compiler find`.
- **`pmix` fails at configure on Debian/Ubuntu with a system zlib**
  (`checking for zlib library (z) in /usr... not found` / `CANNOT
  CONTINUE`). openmpi depends on pmix, so every MPI package is then
  skipped. Cause: spack's `pmix/package.py` passes `--with-zlib=<prefix>`
  but, unlike for libevent and hwloc, no libdir for an external zlib, and
  pmix only searches `<prefix>/lib` and `<prefix>/lib64` — not the
  multiarch `/usr/lib/<triplet>/`. Fix: in
  `repos/builtin/repos/spack_repo/builtin/packages/pmix/package.py`,
  after the `--with-zlib=` line, add
  ```python
  if spec["zlib-api"].external_path:
      dep_libpath = self.find_external_lib_path("zlib-api", "64")
      config_args.append("--with-zlib-libdir=" + dep_libpath)
  ```
  then rerun `spack install`. Revert with
  `git -C repos/builtin checkout .`. Upstream fix belongs in
  spack-packages. As of 2026-10, spack-stack `develop` (the basis for the
  next release) still requires `zlib` and its pmix package is unchanged,
  so the issue is expected to carry forward unless fixed upstream.
  - *Why 1.9 builds didn't hit it:* the pmix code was the same, but 1.9's
    `common/packages.yaml` set `providers: zlib-api: [zlib-ng]`, so spack
    built zlib-ng into its own prefix (`<prefix>/lib`) and ignored the
    system `zlib` external. 2.1 changed this to `zlib-api: require: [zlib]`,
    which matches the system external at `/usr`.
  - *Patch-free alternative for a fresh build:* delete the `zlib` entry
    from `site/packages.yaml` **before the first concretize**, so spack
    builds its own zlib. After packages are built, prefer the patch —
    nearly everything depends on zlib, so changing it rebuilds them.
- **`ewok-env +mysql` provides only the MySQL server** (often the system
  external), not a Python MySQL client. The `py-mysql-connector-python`
  dependency is commented out in the `ewok_env` package. R2D2's own
  `setup.py` lists `mysql-connector-python`, so it arrives when R2D2 is
  pip-installed into the workflow venv. Don't expect
  `import mysql.connector` to work from the stack's Python alone.
- 1.x-era instructions use `packages:all:compiler:[…]` and
  `packages:all:providers:mpi:[…]`; 2.1 CI uses `--compiler` on
  `create env` and `packages:mpi:require:['openmpi@<ver>']`.
