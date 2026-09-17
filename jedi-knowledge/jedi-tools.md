# jedi-tools

**Repository:** https://github.com/jcsda-internal/jedi-tools
**Branch tracked by bundle:** n/a — not a bundle member (clone `develop`)
**Role in JEDI:** shared infrastructure scripts and tooling — environment setup/build scripts, cloud (AWS) utilities, CI and CDash infrastructure, Academy training material, and admin bookkeeping. Not part of the bundle build.

## What it is

`jedi-tools` is a sibling repo to the bundle. It collects the operational
glue around JEDI: per-HPC environment setup scripts, build drivers for
jedi-bundle + the Skylab workflow stack, AWS/cloud provisioning, CI runner
and CDash server configuration, and JEDI Academy teaching material. Images
(`*.png`, `*.jpg`, `*.jpeg`) are stored in **git-lfs** (`.gitattributes`),
mostly under `Academy/`.

## How it fits into the bundle

It does **not** build with the bundle, but `buildscripts/` is the standard
way to build it: `setup.sh` loads the right spack-stack environment for a
host+compiler, then `build_jedi_skylab.sh` builds jedi-bundle and
`build_workflow_apps.sh` builds the workflow stack (simobs, r2d2-client,
ewok, skylab; plus r2d2, r2d2-data, and static-data when
`R2D2_HOST=localhost`). See `jedi-knowledge/workflow.md` for the
workflow stack itself.

## Key directories

- `buildscripts/` — the most-used part of the repo:
  - `setup.sh` — source it to configure a JEDI-Skylab environment. Edit the
    user variables at the top: `JEDI_ROOT`, `HOST`, `COMPILER`,
    `R2D2_USER`, `R2D2_API_KEY`, plus optional `WORKFLOW_ROOT` (defaults to
    `JEDI_ROOT`). Everything else is derived and exported for you —
    `JEDI_SRC=${JEDI_ROOT}/jedi-bundle`, `JEDI_BUILD=${JEDI_ROOT}/build`,
    `JEDI_WORKFLOW=${WORKFLOW_ROOT}/jedi-workflow`, `EWOK_WORKDIR`,
    `EWOK_FLOWDIR`. `source setup.sh -s` loads only the compile
    environment; `-h` for help.
    Header (updated Jan 2026) lists Skylab-capable hosts (localhost,
    derecho, discover-mil, ec2, hercules, orion, s4) vs load-only hosts
    (gaea_c6, hera, narwhal, nautilus, pw-aws/azure/gcloud — several
    marked end-of-support).
  - `setup/<host>_setup_<compiler>.sh` — per-host module/spack-stack load
    scripts that `setup.sh` dispatches to (spack-stack 2.1 era). Valid
    compilers are **`gcc`, `oneapi`, `clang`**; `intel` survives only as
    `derecho_setup_intel.sh` and `hercules_setup_intel.sh` (intel classic
    is deprecated after spack-stack 1.8.0). An `aspire_setup_gcc.sh` exists
    but `aspire` is not yet listed in the `setup.sh` header.
  - `build_jedi_skylab.sh` (`--build-jobs N`, `--help`; auto-estimates the
    job count otherwise), `build_workflow_apps.sh`, `check_in_venv.py`.
  - `compiling/` — older standalone intel build scripts.
- `AWS/` — cloud tooling:
  - `ec2/launch_ec2.py` — launch single EC2 dev instances from
    spack-stack AMIs (plus `system_files/`, `utils/`, tests).
  - `parallel-cluster/` — AWS ParallelCluster config YAMLs (CI, R&D,
    high-res MPAS, skylab-test clusters) and S3 bucket notes.
  - `tools/` — S3 helpers (`presigned_s3_url.py`, `fetch_tempo_s3.sh`,
    `orphaned_snapshots.sh`).
- `Academy/` — JEDI Academy training: `AWS/` fleet launch/suspend scripts
  for classroom instances, `Documentation/` and `Documentation-mini/`
  (Sphinx projects with `push_html.sh`), `Supplemental/activities/`.
- `CDash/` — how to stand up JCSDA's CDash dashboard server
  (docker-compose configs, `cdash_cert_renew.sh`, system-dependency
  install script); pairs jedi-tools configs with the upstream
  Kitware/CDash repo.
- `CI-tools/` — `CI_GitHub_Webhooks.py`, plus `selfhosted/` notes for the
  self-hosted GitHub Actions runners (skylab, simobs, spack-stack
  ubuntu/macOS, ufs-bundle) and Slack integration. Also documents the
  CDash-upload integration used by AWS CodeBuild CI.
- `Data_Repository/` — small Sphinx site generating the "JCSDA Public Data
  Repository" index (`ls_to_sphinx.sh`, `push_html.sh`, `pages/`).
- `ParallelWorks/` — notes for NOAA ParallelWorks cloud clusters (cluster
  config, bootstrap); defers to spack-stack site configs for the generic
  parts.
- `crontabs/` — version-controlled snapshots of admin/system crontabs per
  platform (`<host>/crontab_<user>.txt`); team rule: any live crontab edit
  must be mirrored here.
- `licensing/` — `license.pl` to prepend the Apache license header to
  source files.

## Key entry points

- `README.md` — overview of every top-level directory and full
  documentation of the `buildscripts` variables, plus the convention of
  tagging `develop` before merging buildscript changes that target a new
  spack-stack version.
- `buildscripts/setup.sh` — start here for any "how do I build JEDI on
  machine X" question; the header comment is the authoritative
  platform/compiler support matrix.
- `AWS/ec2/launch_ec2.py` — start here for "give me a JEDI dev box on AWS".

## Common tasks

- **Build JEDI + Skylab on a supported HPC** — copy `buildscripts/setup.sh`
  to `$JEDI_ROOT`, edit the variables, `source setup.sh`, then run
  `bash build_jedi_skylab.sh` and/or `bash build_workflow_apps.sh`.
- **Which compilers work on host X?** — read the header of
  `buildscripts/setup.sh` and check for
  `buildscripts/setup/<host>_setup_<compiler>.sh`.
- **Launch an EC2 instance / cluster** — `AWS/ec2/README.md` and
  `AWS/parallel-cluster/README.md`.
- **CI or CDash questions** — `CI-tools/README.md` (webhook/CodeBuild/CDash
  flow), `CDash/README.md` (server setup).
- **Academy material** — `Academy/Documentation/source/` (needs git-lfs for
  images).

## Gotchas

- **Requires `git-lfs`.** Install it and run `git lfs install` once per
  machine *before* cloning, or the ~60+ image assets come down as pointer
  files.
- `setup.sh` must be **sourced**, not executed, and re-edited per machine —
  it errors out clearly if `<host>_setup_<compiler>.sh` doesn't exist for
  your HOST/COMPILER combination.
- Platform support churns with spack-stack versions; several hosts in the
  setup.sh header are marked "support ended" at specific Skylab releases.
  Pre-merge states are preserved as git tags (see README tagging section),
  so old spack-stack versions are reachable via tags.
- Some content is admin-facing documentation-of-record (crontabs, CDash,
  ParallelWorks) rather than user tooling — the CDash README itself warns
  it may be stale.
- **`gnu` is no longer a compiler name (2026, jedi-tools#501):** every
  `setup/<host>_setup_gnu.sh` was renamed to `..._setup_gcc.sh` and
  `setup.sh` now asks for gcc/oneapi/clang. An existing config with
  `COMPILER=gnu` fails the "no setup script for this HOST/COMPILER" check —
  change it to `gcc`. The same round of cleanup deleted the `jet` and
  `gaea_c5` setup scripts.
- **Build tree now honours `$JEDI_BUILD` (2026-07, jedi-tools#532):**
  `build_jedi_skylab.sh` used to hardcode `cd ${JEDI_ROOT}; mkdir -p build`;
  it now does `mkdir -p ${JEDI_BUILD}` and requires the variable to be set
  (it exits if `setup.sh` wasn't sourced). The default value is unchanged,
  so out-of-the-box behaviour is the same — but you can now point the build
  somewhere other than `${JEDI_ROOT}/build`.
- **R2D2 server localhost install reworked (2026-06, jedi-tools#473):**
  `build_workflow_apps.sh` now treats the `r2d2` repo (the server) as a
  first-class member of `SKYLAB_REPOS` — cloned alongside the others when
  `R2D2_HOST=localhost`, with its branch from `R2D2_BRANCH` (and
  `r2d2-data` now from `R2D2_DATA_BRANCH`). The server repo is cloned but
  not pip-installed; the localhost docker-compose startup flow changed
  too. If you have scripted around the old behavior (the script cloning
  r2d2 itself inside the localhost block), re-check against the new
  script.

## Further reading

- `README.md` in the repo root and the per-directory READMEs.
- `jedi-knowledge/jedi-bundle.md` — what the buildscripts actually build.
- `jedi-knowledge/workflow.md` — the workflow stack installed by
  `build_workflow_apps.sh`.
- https://jedi-docs.jcsda.org/ — `using/jedi_environment/` covers the same
  ground from the user side.
