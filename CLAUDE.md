# CLAUDE.md

This repository is developed WITH BMAD: the plugin is specified by BMAD's planning skills, built story
by story with `bmad-build`, and, as soon as it can, it tracks its own stories in this repository's
issues. Start with `docs/brief-for-bmad.md`: it is the only context the planning was given, and it
lists the decisions that are not open for redesign, the verified facts about BMM 6.12.0 and the
GitLab/GitHub CLIs, and the lessons from the predecessor module.

## Rules

- **Never write the internal GitLab host name** (the one used as a test bed) into any file, commit
  message, issue or PR of this repository. Read it from local config or the environment; say "the
  lab GitLab host" in prose.
- **English** for code, commits, docs and issues. Commits follow `type(scope): description` with the
  five types `feat`, `fix`, `docs`, `chore`, `revert`; the description states the effect or the
  symptom, never the operation; issue references only in the footer (`Refs #n` / `Closes #n`).
- Planning artefacts under `_bmad-output/` are committed: they are the specification. Generated
  BMAD state (`_bmad/render/`, `_bmad/scripts/`, per-module config) is not — see `.gitignore`.
- Nothing is merged without its tests. Anything that talks to GitLab or GitHub is also exercised
  against a real project of that platform before it is merged.

## Reference

The predecessor module and its full test lab live locally at
`../bmad-issue-tracking` (public fork: https://github.com/pasqualtroncone/bmad-issue-tracking, branch
`devel`). Read its `CLAUDE.md` for the hook model, title formats, platform differences and the CI
state rules; read its issues #1–#105 for the defect history. Do not copy its code.

The Claude Code session that ran that lab is available as a reference for questions: on this
machine it appears in `ListAgents` as `bmad-issue-tracking-e2e-lab` and can be messaged with
`SendMessage`. Ask it about anything in the brief that needs more evidence.
