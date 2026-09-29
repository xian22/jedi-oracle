# JEDI coding practices

Team-curated guidelines for writing, reviewing, and modifying JEDI code.
Applies to human contributors and AI agents alike. Add anything the team
wants consistently applied — patterns that have burned us, conventions the
style guide doesn't capture, or rules that AI assistants tend to get wrong.

This file is tracked. Open a PR to contribute.

---

## General principles

- **Understand before you change.** Read the relevant code, tests, and docs
  before proposing or making edits. Don't guess at behavior — verify it.
- **Minimal footprint.** Only change what the task requires. Don't refactor
  surrounding code, rename unrelated identifiers, or add abstractions the
  task doesn't call for.
- **Plan before you edit across repos.** Any change that touches more than
  one JEDI repo needs a written plan agreed on before files are modified.
  State which repos, which interfaces, and in what order changes should land.
- **Tests are not optional.** New behavior gets a test. Bug fixes get a
  regression test. If a test is impractical to add, say why.
- **Prefer generic, reusable code.** JEDI is a multi-model, multi-application
  framework — code that only works for one model, one observation type, or
  one configuration is a liability. Before adding something model- or
  application-specific, ask whether it belongs in a lower layer (OOPS, UFO,
  SABER) as a general capability. Prefer parameterization over duplication,
  and template or abstract base patterns over copy-paste specialization.

---

## C++ conventions

- Follow the existing style in the file you're editing — JEDI repos use
  clang-format; don't reformat lines you haven't changed.
- Prefer `const` references for function parameters that are not modified.
- Use OOPS abstractions (`ObsSpace`, `State`, `Increment`, etc.) rather than
  reaching through them to underlying data structures.
- Forward-declare where possible; avoid unnecessary `#include` in headers.
- Template instantiations live in `*_.cc` or explicit instantiation files —
  don't add them to headers without a reason.

---

## Fortran conventions

- New code should be Fortran 2003 or later; avoid Fortran 77 idioms.
- Declare intent (`intent(in)`, `intent(inout)`, `intent(out)`) on all
  dummy arguments.
- Use `implicit none` in every module and program unit.
- Prefer module procedures over standalone subroutines for anything that
  operates on a derived type.

---

## Python conventions

- Follow PEP 8. Use type hints on public functions.
- Don't vendor dependencies — use what spack-stack already provides.
- Scripts that wrap ecflow or skylab tasks should be idempotent where
  possible.

---

## YAML / configuration

- Keep YAML configs self-documenting: prefer explicit keys over positional
  defaults, and add inline comments for non-obvious values.
- Don't hard-code machine-specific paths; use environment variables or
  spack-stack variables instead.
- When modifying a test YAML, check whether other tests share it via
  symlink or `include` before changing values.

---

## Automated coding norms

Every JEDI source repo enforces coding norms through a CTest that must pass
before a PR can merge. Understanding how these work saves debugging time.

### C++ — cpplint

- **Tool:** Google's `cpplint.py` (each repo vendors a copy in `tools/`)
- **Standard:** [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- **Line length:** 100 characters (set in `CPPLINT.cfg` at repo root and `src/`)
- **Test name pattern:** `{repo}_coding_norms` (e.g. `ufo_coding_norms`,
  `ioda_coding_norms`, `soca_coding_norms`)
- **Run it:** `ctest -R <repo>_coding_norms` from your build directory
- **Labels:** Many repos mark this test `tier2` — it still needs to be green
  before merge, but it may not run by default with a bare `ctest`.

Common cpplint filter flags applied across repos:

```
filter=+build,+legal,+readability,+runtime,+whitespace,
       -runtime/references,-runtime/printf
```

OOPS and VADER additionally disable `-build/c++11`.

### C++ — clang-format

- **Config file:** `.clang-format` at repo root (Google style base, 100-column limit,
  2-space indent)
- IODA and UFO have the most detailed configs; others inherit Google defaults.
- Run `clang-format -i <file>` to auto-fix formatting before committing.
- Don't reformat lines you haven't changed — it pollutes the diff and makes
  review harder.

### Python — pycodestyle

- **Tool:** `pycodestyle` (formerly pep8)
- **Config file:** `.pycodestyle` at repo root
- **Line length:** 120 characters
- Common ignores: `E741, W605, W291, E722, W503, W504`
- IODA also runs `ioda_pylint.sh` for broader Python lint checks.
- Run `pycodestyle --config=./.pycodestyle .` from the repo root.

### Practical tips

- Run the coding norms test locally before pushing — CI rejections for style
  are easy to avoid and slow down review cycles.
- If `ctest -R <repo>_coding_norms` fails, the output will list the offending
  file and line. Fix those lines only; don't bulk-reformat the file.
- New files need a `CPPLINT.cfg` in their directory only if they need to
  override root settings (e.g. a different line length for generated code).
- The workflow repos (skylab, ewok, simobs, r2d2) are configuration/scripting
  repos — no cpplint test exists there, but Python style still applies.

---

## Pull requests

- One logical change per PR. If a refactor is needed to land a feature,
  send the refactor first as a separate PR.
- PR descriptions should explain *why*, not just *what* — the diff shows
  what changed; the description should say why it had to.
- Tag the relevant repo owners and anyone whose interface you changed.
- CI must be green before requesting review. Don't ask reviewers to ignore
  failing checks.

---

## For AI agents specifically

- **Don't invent interfaces.** If you're unsure whether a function or class
  exists, search the cloned source before using it.
- **Don't silently skip tests.** If the test suite fails, report it — don't
  remove or disable the test to make the build pass.
- **Surface uncertainty.** If a change has a non-obvious impact on another
  repo (e.g., a changed OOPS interface affects all model-specific repos),
  flag it explicitly rather than assuming the downstream is unaffected.
- **Prefer small, reviewable diffs.** A 50-line PR that does one thing is
  easier to review than a 500-line PR that does five. Split where natural.
- **Don't credit a specific AI in commits or PRs.** No AI `Co-Authored-By`
  trailers, session links, or "Generated with <tool>" lines. When generative
  AI contributed, end the commit message or PR description with:
  `Generative AI was used in preparing this change.`
- **Read this file before planning any code change.** The practices here
  represent team consensus — don't override them without a stated reason.
