# jedi-docs

**Repository:** https://github.com/jcsda-internal/jedi-docs
**Branch tracked by bundle:** n/a — not a bundle member (clone `develop`)
**Role in JEDI:** the source for the rendered JEDI documentation site. Sphinx-based; not part of the bundle build.

## What it is

`jedi-docs` is the documentation source repo: two independent Sphinx
projects (`docs/` — the main "JEDI Documentation", and `jedi-edu/` — the
"JEDI-EDU Manual") plus a directory of unrendered markdown how-tos
(`howto/`). The main site is published via Read the Docs
(`.readthedocs.yml`: ubuntu-22.04, Python 3.11, builds `docs/conf.py`) at
https://jointcenterforsatellitedataassimilation-jedi-docs.readthedocs-hosted.com
(reachable via https://jedi-docs.jcsda.org/).

Pages are mostly reStructuredText; `myst_parser` is enabled so Markdown
works too. Images (`*.png`, `*.jpg`) are stored in **git-lfs** (see
`.gitattributes`), so git-lfs must be installed to get them.

## How it fits into the bundle

- **Not built by the bundle.** No `ecbuild_bundle(...)` entry. It's a
  sibling resource cloned by the oracle's init flow on request.
- **Consumed by:** humans, and the oracle (read the source to answer
  questions). `docs/inside/jedi-components/` mirrors the bundle's
  component repos one-to-one.

## Key directories

There is no `source/` subdirectory — `.rst` files live directly under
`docs/`. Topic map (where to look when a user asks about X):

- `docs/overview/` — what JEDI is, methodology, governance, requirements.
- `docs/using/` — user-facing how-to-run docs:
  - `building_and_running/` — `building_jedi.rst`, `running_jedi.rst`,
    `config_content.rst` (YAML configuration content).
  - `jedi_environment/` — `modules.rst`, `spackbuild.rst`, `cloud/`,
    `containers/` (environments, spack-stack, containers, cloud).
  - `running_skylab/` — `running_skylab.rst`, `HPC_users_guide.rst`.
- `docs/inside/` — developer-facing internals:
  - `jedi-components/` — per-repo deep dives: `oops/` (introduction,
    `applications/`, `algorithmic_details/`, `interfaces/`,
    `generic-implementations/`, `toy-models/`), `ufo/` (`qcfilters/` incl.
    obsfunctions, `obsops.rst`, `obserrors/`, `variabletransforms/`,
    `varbc.rst`, `newobsop.rst`, `dataflow.rst`, `marineufo.rst`), plus
    `ioda/`, `saber/`, `vader/`, `fv3-jedi/`, `mpas-jedi/`, `soca/`,
    `skylab/`, and `configuration/`.
  - `developer_tools/` — cmake, debuggers, doxygen, gcov, gitflow, sphinx,
    **gitlfs.rst**, gptl, homebrew, oops-env-variables.
  - `practices/` — gitflow, pull requests, issues, openmp, ecmwf_forks,
    documenting.
  - `testing/` — `adding_a_test.rst`, `unit_testing.rst`.
  - `conventions/` — naming/layout conventions (markdown).
- `docs/working-practices/` — fork/clone, branch/merge, code review,
  testing, creating docs.
- `docs/FAQ/`, `docs/ref/` (`refs.bib` bibliography), `docs/learning/`
  (currently a stub pointing at the 1.3.0 release docs and past JEDI
  Academies).
- `jedi-edu/` — separate Sphinx project (variational / ensemble / hybrid DA
  educational material); built and pushed manually (`push_html.sh`).
- `howto/` — unrendered markdown working notes: `macos/` (minimum setup,
  VS Code debugging), `profiling/` (profiling.md + tool screenshots),
  `cylc/` (cylc-on-mac recipes).

## Key entry points

- `docs/index.rst` — root toctree of the main site.
- `docs/conf.py` — extensions (`sphinxcontrib-bibtex`, `myst_parser`,
  `sphinx_rtd_theme`), version (currently `8`), and a custom
  `ultimate_replacements` substitution mapping `{skylab_version}` →
  `skylab-8.0.0` and `{skylab_v}` → `Skylab v8` (must be bumped at release).
- `docs/requirements.txt` — pinned build deps (`sphinx_rtd_theme==0.5.2`,
  `docutils==0.16`, `urllib3<2`).
- `.readthedocs.yml` — hosted-build definition.
- `README.md` — contribution workflow and reST style rules (use `:ref:`
  with `.. _handle:` anchors, never relative file paths).

## Common tasks

- **Preview locally** — `cd docs && make html`, open
  `_build/html/index.html` (install `docs/requirements.txt` first; on
  JCSDA-supported machines `module load jedi-tools-env` provides it).
  Same recipe in `jedi-edu/`.
- **Find a topic** — `grep -ri <term> docs/ jedi-edu/ howto/`; or use the
  topic map above. Component questions almost always land in
  `docs/inside/jedi-components/<repo>/`.
- **Add a page** — drop an `.rst` (or `.md`) under the right section and
  add it to that section's `index.rst` toctree; cross-reference with
  `:ref:` handles, not file paths.
- **Check what changed recently** — `git -C jedi-docs log --oneline`;
  recent activity is dominated by UFO QC-filter/ObsFunction pages and
  oops ensemble-DA (EnKF/ETKF) algorithm docs.

## Gotchas

- **New pages (2026-09):** `saber/blocks/BUMP_nicas_estimation.rst`
  (BUMP HDIAG, jedi-docs#1085) and
  `ufo/qcfilters/obsfunctions/PotentialTemperatureFromTemperature.rst`
  (jedi-docs#883). Also landed: TLAD equations for the AOD mass-fraction
  UFO operator (#1079), a variable-groupings example for the diffusion
  block (#1096), and vertical localization for the sequential EnKF
  (#1031).
- **The docs lag the code.** jedi-docs is updated by hand and is often a
  release or more behind what's in the cloned source repos. Treat it as
  orientation, then verify behavior/options against the actual code (and
  each repo's in-tree docs/tests) before answering precisely.
- Two separate Sphinx projects (`docs/`, `jedi-edu/`) build independently;
  cross-references between them need intersphinx or explicit URLs.
- `howto/` is not rendered into the public site — it's working notes only.
- Images are git-lfs pointers without `git lfs install`; a clone without
  lfs builds but shows broken images.
- `docs/venv/` is a directory tracked in the repo — ignore it; don't use it
  as your build venv.
- The `{skylab_version}` substitution in `conf.py` means literal
  `{skylab_version}` strings in `.rst` get rewritten at build time — check
  `conf.py` before tagging a release.

## Further reading

- Published site: https://jedi-docs.jcsda.org/
- In-repo: `docs/index.rst`, `docs/inside/jedi-components/`, `howto/`.
- Related briefs: every `jedi-knowledge/<repo>.md` cites jedi-docs for
  canonical user-facing documentation.
