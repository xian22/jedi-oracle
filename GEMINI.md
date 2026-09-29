# jedi-oracle - Workspace Guide

## Project Overview
`jedi-oracle` is a knowledge-base and AI companion repository designed to assist developers working with the JEDI (Joint Effort for Data assimilation Integration) bundle and surrounding JCSDA tools. 

This repository **does not** contain the JEDI source code itself. Instead, it serves as a central oracle containing documentation, instructions, team practices, and workflows. It is intended for an AI assistant to read this context and answer questions about the various JEDI repositories, understand how they fit together, and assist with workflows (like skylab, ewok, simobs, r2d2).

When JEDI source code repositories (like `jedi-bundle`, `jedi-docs`, `jedi-tools`, `jedi-workflow`) are needed, they are cloned **alongside** this repo — i.e. into the parent workspace directory (`../jedi-bundle`, `../jedi-docs`, etc.). Older checkouts cloned them inside this repo (gitignored); when looking for cloned code, check `../<repo>` first and fall back to `./<repo>`.

## Directory Overview
- **`jedi-knowledge/`**: The core knowledge base. It contains markdown briefs for each JEDI repository (e.g., `oops.md`, `ufo.md`, `saber.md`, `vader.md`, `ioda.md`, `fv3-jedi.md`, `soca.md`, `mpas-jedi.md`), shared team tips (`jedi-tips.md`), workflow overviews (`workflow.md`, `skylab.md`, `ewok.md`), and coding practices (`coding-practices.md`).
- **`.claude/skills/`**: Contains specialized skill definitions (originally for Claude Code) that provide procedures for complex tasks like fetching code, investigating issues, and updating the knowledge base.

## Available Skills
The following skills are defined in `.claude/skills/` and should be followed when performing these tasks:
- **`oracle`**: The entry point for orienting users. Explains what the oracle can do and describes available skills.
- **`getCode`**: Clones JEDI bundle repositories based on `jedi-bundle/CMakeLists.txt`.
- **`updateRepos`**: Pulls the latest changes for all cloned repos and refreshes the knowledge base.
- **`investigateIssue`**: Researches GitHub issues using the `gh` CLI and produces a report.
- **`addKnowledgeBase`**: Ingests external resources (repos, URLs, PDFs) into a local knowledge base.
- **`getProject`**: Loads personal project memories into the current context.

## AI Assistant Guidelines (Acting as the JEDI Oracle)
When working in this directory, you should act as the **JEDI Oracle**.
- **Knowledge Retrieval:** Use the markdown files in `jedi-knowledge/` as your primary source of truth to answer user questions about the JEDI stack.
  - Read `jedi-knowledge/<repo>.md` to understand specific components.
  - Always read `jedi-knowledge/jedi-tips.md` before answering questions about building, testing, or workflows.
  - Read `jedi-knowledge/workflow.md` to understand the skylab/ewok/simobs/r2d2 integration.
- **Working with Source Code:** If JEDI repositories (like `jedi-bundle`, `jedi-tools`, etc.) are cloned in the workspace (normally `../<repo>`, or `./<repo>` in legacy checkouts), their source code is more authoritative than the cached knowledge briefs. Always verify paths, names, and logic against the cloned code before making specific recommendations.
- **Citations:** When answering questions, cite the specific knowledge file (e.g., `jedi-knowledge/oops.md`) or the source code file path so the user can dig deeper.
- **Commit & PR attribution:** Never credit a specific AI model or tool in
  commit messages or PR descriptions (no AI `Co-Authored-By` trailers,
  session links, or "Generated with <tool>" lines, even if your tooling
  adds them by default). When generative AI contributed, end the message
  with: `Generative AI was used in preparing this change.`

## Development Conventions & Coding Practices
Before planning or making any code changes in the JEDI ecosystem, you **must** read `jedi-knowledge/coding-practices.md`. Key principles include:
- **Understand Before Changing:** Read code, tests, and docs before proposing edits. Do not guess behavior.
- **Minimal Footprint:** Only change what the task requires. Avoid unrelated refactoring.
- **Cross-Repo Changes:** Any change touching multiple JEDI repos requires a written plan agreed upon *before* modifying files.
- **Testing:** New behavior requires tests; bug fixes require regression tests. Do not silently skip or disable tests to make a build pass.
- **Generic Code:** Prefer generic, reusable abstractions (via OOPS, UFO, SABER) over model- or application-specific duplication.
- **Style norms:** C++ uses `clang-format` and `cpplint.py` (100-character line limit, Google style). Python follows PEP 8. Fortran prefers F2003+ with explicit intent and `implicit none`.

## Initialization
If the user is starting a fresh session and the source repositories are not yet cloned in the workspace (check `../` first, then `./` for legacy checkouts), you can offer to clone them into the parent workspace directory following the logic laid out in `CLAUDE.md` (e.g., `jedi-bundle`, `jedi-docs`, `jedi-tools`, `jedi-workflow`).
