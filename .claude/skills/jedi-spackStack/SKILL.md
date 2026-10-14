---
name: jedi-spackStack
description: Guide the user through building a spack-stack environment for JEDI on a machine that is not a supported platform (a newer or different Linux distribution, macOS, a workstation). Determines the right spack-stack release and compiler from jedi-tools and spack-stack itself, derives prerequisites and settings from that release's CI recipe, and verifies the concretized environment before the long install.
---

# /jedi-spackStack

Build a spack-stack environment for JEDI on a machine that spack-stack does
not officially support. The skill works out *which* versions to use from
the cloned sources, adapts the release's CI recipe to the local
machine, and catches silent problems at concretize time — before a
day-long install.

**Read `jedi-knowledge/spack-stack.md` first.** It explains the config
layers, how compilers and externals are registered, how the solver picks
variants, and the per-release known issues. Also read the spack-stack
section of `jedi-knowledge/jedi-tips.md`.

## Principles

- **Derive, don't hardcode.** MPI versions, `external find` exclusions,
  variant settings and module paths change between releases. Read them
  from the target release's CI workflow and configs at run time. Commands
  copied from older docs pages are a common source of breakage.
- **The user runs anything needing root** (package installs,
  `update-alternatives`). Give them one block to paste, then verify the
  result yourself.
- **Shell state does not persist** between your commands. Every spack
  command block must re-run `source setup.sh` and `spack env activate`.
- **Back up before editing** `site/packages.yaml` or `spack.yaml`
  (`cp file file.orig`), and tell the user where the backup is.
- **Don't start `spack install` yourself.** It runs for hours to days;
  hand the user the command to run under `screen` or `tmux`.
- **Be clear about what is tested.** If the target platform has no CI
  workflow or site config in the release, say the path is docs-guided.

## Flow

### 1. Detect the platform

Gather, without asking where you can detect it:

- OS and version (`/etc/os-release`, or `sw_vers` on macOS), architecture.
- Package manager (apt, dnf/yum, zypper, brew).
- Core count (`nproc` / `sysctl -n hw.ncpu`) and free disk where the stack
  will live. The largest templates need well over 100 GB; few cores mean
  a multi-day install — say so.
- Compilers installed, and **compiler versions the package manager
  offers** (e.g. `apt-cache search --names-only '^gcc-[0-9]+$'`,
  `dnf list 'gcc-toolset-*'`, `brew search gcc llvm`).
- Whether `module` works in a login shell, and `ulimit -s`.

### 2. Choose the target

Ask the user:

- **What JEDI are you building against?** `develop` or a tagged release.
- **Which template?** `skylab-dev` (JEDI + Skylab, smaller) or
  `unified-dev` (everything, largest). Others exist in
  `configs/templates/` of the release.
- **Local R2D2 (MySQL)?** Only needed to run Skylab with a local database.

### 3. Make sure the sources are cloned

Resolve the workspace root per CLAUDE.md. Needed:

- **jedi-tools** (`<workspace>/jedi-tools/`) — the per-host setup scripts
  are the best record of which spack-stack release and compiler versions
  JEDI currently runs with. If missing, offer the clone from CLAUDE.md's
  init flow (needs git-lfs).
- **jedi-docs** (optional) — release table and `spackbuild.rst`, useful
  but often behind.
- **spack-stack** at the chosen release, at
  `<workspace>/spack-stack/spack-stack-<version>/`:
  ```bash
  git ls-remote --heads --tags https://github.com/JCSDA/spack-stack.git
  git clone -b <tag-or-release-branch> --recurse-submodules \
      https://github.com/JCSDA/spack-stack.git <workspace>/spack-stack/spack-stack-<version>
  ```
  spack itself comes in as a submodule. jedi-bundle is **not** needed for
  this — it pins neither compilers nor spack-stack.

Pick the release: for `develop`, the release used by the jedi-tools
setup scripts (`grep -h 'spack-stack-[0-9]' <jedi-tools>/buildscripts/setup/*.sh`)
— normally the newest. For a tagged JEDI release, the paired release from
the jedi-docs table.

### 4. Choose the compiler

- From the jedi-tools setup scripts, list the compiler versions supported
  hosts use with that release (`module load stack-<compiler>/<version>`
  lines).
- Pick the closest version the local package manager offers in that
  family. Prefer it over the OS default when the OS default is a newer
  major than any supported host uses.
- Linux → `gcc` unless the user has a reason otherwise. macOS → `clang`;
  read the release's `configs/common/packages_clang.yaml` to see which
  toolchain it expects (2.1 prefers `llvm` for all three languages).
- Present the choice with the evidence (which hosts, which versions).

### 5. Prerequisites (user runs, needs root)

Build the list from, in order of authority:

1. The closest site config README in `configs/sites/tier2/` (e.g. an
   Ubuntu or RHEL site) — it has the OS package list.
2. The release's CI workflow in `.github/workflows/` — what the CI
   machines needed.
3. The spack-stack docs "New site configs" prerequisites for the release
   (fetch, or ask the user to paste it).

Translate package names to the local package manager and **verify each
exists** (`apt-cache policy`, `dnf info`, `brew info`) — drop or replace
any that don't (note which). Always include: the chosen compiler's C, C++
and Fortran packages, environment modules (tcl `environment-modules` on
Linux unless the site uses lmod), git, git-lfs, build tools, curl/ssl
headers, Qt 5 dev packages if the template's ecflow needs `+ui` (check
`common/packages.yaml`), MySQL server + client headers if the user wants
local R2D2.

If the chosen compiler is not the OS default, include
`update-alternatives` (Linux) to make it the default, and tell the user
how to undo it.

Give the user one block to paste. Ask them to log out and back in if
environment modules were just installed. Then verify: compiler versions,
`module --version` in a login shell, MySQL if requested.

### 6. Create the environment and register the system

Use the release's CI workflow as the template for this step — copy its
sequence, adjusting only names. Typically:

```bash
cd <spack-stack-dir> && source setup.sh
spack stack create env --site <linux.default|macos.default> \
    --template <template> --name <env-name> --compiler <compiler>
spack env activate envs/<env-name>
unset SPACK_DISABLE_LOCAL_CONFIG
export SPACK_SYSTEM_CONFIG_PATH="$PWD/envs/<env-name>/site"
spack external find --scope system <exclusions from the CI workflow>
spack external find --scope system <individual packages from the CI workflow>
spack compiler find --scope system
export SPACK_DISABLE_LOCAL_CONFIG=true
unset SPACK_SYSTEM_CONFIG_PATH
```

Name the env following the supported-host convention
(`ue-<compiler>-<version>` for unified, `se-…` for skylab).

Then review `site/packages.yaml`:

- **Remove every compiler except the chosen one** (back up first). OS
  default compilers are often detected even when not the default.
- On multiarch distributions (Debian/Ubuntu), check the known issues for
  externals whose libraries live in `/usr/lib/<triplet>/` — e.g. 2.1's
  `zlib` external breaks `pmix`. Decide before the first concretize:
  dropping an external later rebuilds everything that depends on it.
- Note externals much newer than the release was tested with — don't
  remove them yet, but remember them for step 9.
- A umask warning is harmless for a single-user install.

### 7. Apply the release's settings

Copy the `spack config add` lines from the CI workflow (MPI provider,
template-specific variants), skipping CI-only items (build caches,
`padded_length`, mirrors). Add `packages:ewok-env:require:['+mysql']` if
the user wants local R2D2. Check `jedi-knowledge/spack-stack.md` "Known
issues by release" for anything to apply up front.

### 8. Concretize and verify

Run in the background (it can take tens of minutes for large templates):

```bash
spack concretize --force --fresh 2>&1 | tee log.concretize
${SPACK_STACK_DIR}/util/show_duplicate_packages.py <-i flags from CI> log.concretize
spack stack check-preferred-compiler 2>&1 | tee log.check-preferred-compiler
```

Then check, and report each item to the user:

- **Compiler:** no package uses an unintended compiler (grep the log for
  the removed versions; read the check output for anything not "uses
  preferred compiler" / "has no compiler dependency").
- **MPI:** exactly the intended provider and version.
- **Duplicates:** none beyond those deliberate in the template.
- **Expected features are present.** Read `spack.lock` and compare key
  root packages' variants against their defaults — at minimum `ewok-env`
  (`ecflow`, `mysql`) and anything the user specifically needs. The
  solver drops default-on variants silently when keeping them would need
  a deprecated version; a `deprecated version` warning in the log is the
  clue. Diagnose and fix as in `jedi-knowledge/spack-stack.md`, then
  re-concretize.
- Count of unique specs (`len(concrete_specs)` in `spack.lock`), to set
  expectations.

### 9. Hand off the install

Give the user the block to run under `screen` or `tmux`:

```bash
cd <spack-stack-dir> && source setup.sh
ulimit -S -s unlimited
spack env activate envs/<env-name>
cd envs/<env-name>
spack install --verbose 2>&1 | tee log.install
```

Tell them: reruns resume where they stopped; network timeouts look like
failures and usually pass on rerun. When a package fails repeatedly, ask
for the tail of `log.install` and the package's build log (spack prints
its path, ending in `spack-build-out.txt`, with the error). Common fixes on
unsupported platforms:

- Check "Known issues by release" in `jedi-knowledge/spack-stack.md`
  first — e.g. `pmix` failing to find a system zlib on Debian/Ubuntu.
- Failure inside a too-new system tool → delete that external from
  `site/packages.yaml`, re-concretize, rerun.
- Failure from a too-new compiler or header → check the compiler actually
  used; spack may have picked a different one.

### 10. Modules

Confirm the install finished first. Don't rely on grepping the log for
"Error" (see `jedi-knowledge/spack-stack.md`). Check that every
non-external spec in the environment is installed, using a short
`spack python` script. Then (any shell, with `setup.sh` sourced and the
env activated — not only the install's screen session):

```bash
spack module tcl refresh -y     # or lmod if the site uses lmod
spack stack setup-meta-modules
```

Verify in a fresh login shell, loading **one level per command**
(the hierarchy only appears as each meta-module loads):

```bash
module use <spack-stack-dir>/envs/<env-name>/modules/Core
module load stack-<compiler>/<version>
module load stack-<mpi>/<version>
module load <env modules, e.g. jedi-fv3-env ewok-env>
which mpicc python3 ecbuild ecflow_client
```

Check that the tools resolve inside the env's `install/` tree, not `/usr`.
Point the user to `/jedi-getCode` and jedi-docs' build instructions for
the next step. Mention that to use local R2D2, the MySQL Python client
comes from pip-installing R2D2, not from the stack.

## Recording what was learned

When the run uncovers a release-specific problem and fix, offer to add it
to "Known issues by release" in `jedi-knowledge/spack-stack.md` with the
month observed — **generic wording only**: no hostnames, usernames, local
paths or project names (see CLAUDE.md's user-specific content rule).
Machine-specific notes go in `jedi-knowledge/project-notes/` (gitignored).
