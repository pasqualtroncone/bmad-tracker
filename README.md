# bmad-tracker

A [BMAD](https://github.com/bmad-code-org/BMAD-METHOD) module that makes the issue tracker
(GitLab or GitHub) the source of truth for a BMAD sprint, so that several people and agents can
work on the same project in parallel: one worktree and one merge request per story, a CI gate
before anything is marked done, and the tracker mirrored back into the files BMAD reads.

**Status:** greenfield. This repository is being specified, built and tested with BMAD itself.
The planning artefacts under `_bmad-output/` are the living specification; `docs/brief-for-bmad.md`
is the context the planning started from.

## Origin

`bmad-tracker` is based on the idea and the experience of
[**bmad-issue-tracking** by Jerome Revillard](https://github.com/jrevillard/bmad-issue-tracking) (MIT),
which mirrors BMAD sprint tracking to GitLab and GitHub issues through TOML hooks on the BMM
workflows. Four days of end-to-end testing of that module against real GitLab and GitHub
repositories produced the lessons this project starts from — see `docs/brief-for-bmad.md`. The
hook model, the issue and MR title formats and the CI-gate contract are inherited from it; the
engine and the source-of-truth model are new.

## License

MIT — see `LICENSE`.
