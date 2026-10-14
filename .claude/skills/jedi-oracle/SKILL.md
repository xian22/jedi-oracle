---
name: jedi-oracle
description: Entry point for the JEDI oracle. Reads CLAUDE.md and gives the user a description of every available skill. Use this when the user wants to know what the oracle can do, how to get started, or which sub-skill to reach for.
---

# /jedi-oracle

The user-facing entry point for the JEDI oracle. When the user types
`/jedi-oracle`, this skill orients them: it reads `CLAUDE.md` for the
operating context, then describes each sub-skill so the user knows
what to invoke next.

This is the right command to suggest when:

- A new user has just opened the repo and isn't sure where to begin.
- Anyone asks "what can the oracle do?" or "what skills are available?"
- The auto-intro from `CLAUDE.md`'s first-load path didn't fire (this
  skill is the explicit fallback).

## Flow

1. **Read `CLAUDE.md`** at the oracle repo root. This loads the oracle's
   operating context: the init flow, the layout, and the list of
   knowledge files.

2. **Greet briefly.** One short paragraph explaining what the oracle
   is and what it knows about (the JEDI bundle, jedi-docs, jedi-tools,
   the workflow stack of skylab/ewok/simobs/r2d2, and the
   user-curated knowledge base).

3. **Describe each sub-skill.** Use the structure below. Read each
   sub-skill's `SKILL.md` first if you need to refresh on details —
   don't paraphrase from memory.

   - **`/jedi-getCode`** — *Clone bundle repositories.* Parses
     `jedi-bundle/CMakeLists.txt`, asks which repos to clone (all,
     whitelist, or blacklist) and which optional packages to enable
     (RTTOV, OASIM, ROPP, GSIBEC, IODA-converters, PyIRI), and clones
     the selected repos into `jedi-bundle/<repo>/`.

   - **`/jedi-updateRepos`** — *Pull develop and refresh the knowledge
     base.* For every cloned repo (the bundle sub-repos plus
     `jedi-bundle/`, `jedi-docs/`, `jedi-tools/`, and the
     `jedi-workflow/` repos if present), pulls `develop`, summarizes
     the diffs to `whatHasChanged.md`, and asks whether to also
     update the per-repo briefs in `jedi-knowledge/` to match.

   - **`/jedi-addKnowledgeBase`** — *Extend the oracle with an external
     resource.* Ingests another GitHub repo, a web page, a local
     PDF, or any other reference the user provides. Writes a brief
     under `jedi-knowledge/external/<name>.md` and indexes it in
     `externalKnowledge.md` (gitignored — local to the user).

   - **`/jedi-getProject`** — *Load personal project context.* Reads the
     user's Claude memory files, filters for `type: project` entries,
     lists them, and loads whichever the user selects into the current
     conversation so the oracle is aware of active work without the user
     having to re-explain it.

   - **`/jedi-investigateIssue`** — *Research a GitHub issue.* Asks for
     keyword(s), searches issues you're involved in across all repos
     (default `--involves=@me`), lets you pick from the matches, then
     pulls the full issue and discussion with `gh` and writes a report
     covering the problem, what the code shows, and possible solutions.

   - **`/jedi-spackStack`** — *Build spack-stack on an unsupported
     machine.* Detects the platform, picks the spack-stack release and
     compiler from jedi-tools' setup scripts, derives prerequisites and
     settings from that release's CI recipe, and checks the concretized
     environment (compiler, MPI, duplicates, silently dropped variants)
     before handing over the long `spack install`.

4. **Check the workspace state.** Resolve the workspace root per
   CLAUDE.md (`../` in the sibling layout, the oracle root in legacy
   nested checkouts). If `jedi-bundle/`, `jedi-docs/`, `jedi-tools/`, or
   `jedi-workflow/` are missing there, mention this and offer to walk
   through the init flow from `CLAUDE.md` (clone the missing pieces —
   new clones go alongside jedi-oracle at `../`). If everything is
   already in place, skip this and go straight to the next step.

5. **Invite the next step.** End with a short prompt like *"What
   would you like to work on?"* or *"Want me to start with one of
   the skills above?"*

## Notes

- This skill's job is orientation, not execution. Do **not** invoke
  the sub-skills directly — let the user pick.
- Keep the descriptions short. The full `SKILL.md` for each sub-skill
  has the depth; this is the elevator pitch.
- Don't reprint the entire `CLAUDE.md`. Reading it gives you context;
  the user just needs the highlights.
- After running `/jedi-oracle` once in a conversation, don't repeat the
  full overview unless the user asks.
