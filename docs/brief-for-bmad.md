# Brief for BMAD: what `bmad-tracker` must become, and what we already know

This is the context handed to BMAD's planning skills (`bmad-product-brief`, `bmad-prd`,
`bmad-architecture`, `bmad-create-epics-and-stories`, `bmad-sprint-planning`). It contains
**decisions** (taken by the product owner, not open for redesign), **verified facts** about the
environment, **lessons** from the predecessor module, and **process requirements**. It deliberately
contains **no design**: no command list, no data model, no epic breakdown. Those are BMAD's to
produce. When the planning needs something that is not here, it asks; the answer will be a fact
or a decision, never a design.

Written 2026-09-22 by the team that spent four days testing the predecessor end to end.

---

## 1. Problem and goal

BMAD (BMM 6.12) plans and builds software through skills that write files: a PRD, an architecture
spine, `epics.md`, `sprint-status.yaml`, one `spec-<key>.md` per story. Those files are the only
state BMAD has. That makes BMAD excellent for one person or one agent, and hostile to **parallel
work**: two people building two stories of the same sprint edit the same `sprint-status.yaml` on
the same branch, nobody can see who is working on what, and `bmad-loop` (the unattended runner)
has to be the single writer of the file to stay coherent.

The predecessor, `bmad-issue-tracking` by Jerome Revillard, mirrored those files **one way** into
GitLab/GitHub issues (PRD → issue, epic → issue, story → issue with a status label, story branch →
merge request, CI gate before "done"). It proved the hook model works and that the tracker is the
right shared surface — and it also proved that a one-way mirror of a file does not unlock
parallel work, and that its execution engine (a YAML pseudocode interpreted by the coding agent)
cannot be tested without a live LLM.

**Goal.** `bmad-tracker` makes the **tracker the source of truth** of a BMAD sprint: a story's
state, owner, branch, merge request and CI verdict live in the issue tracker; the files BMAD reads
are derived from it. Several people and agents work on the same sprint in parallel, each story in
its own worktree and merge request, and nothing is marked done without a green CI on the pushed
commit. It works with any repository that depends on BMAD — it is not built for one product.

## 2. Decisions already taken (product owner: Pasqual Troncone)

| Topic | Decision |
|---|---|
| Name and hosting | `bmad-tracker`. Public GitHub repository `pasqualtroncone/bmad-tracker`, MIT license. |
| Origin | The README credits `jrevillard/bmad-issue-tracking` as the origin of the idea and of the lessons. Nothing of its code is copied; it is reference reading. |
| Engine | A **deterministic CLI in Python, run with `uv`** (already a BMM prerequisite). BMAD's TOML hooks invoke that CLI. The coding agent contributes **text** only where language is needed (an issue description, a review summary) and hands it to the CLI as a file. Every action the CLI takes against git or the tracker must be inspectable before it runs (a dry-run that prints the exact commands), because inspectability was the predecessor's design goal and it is worth keeping. |
| Source of truth | The **tracker** (issues), from version 1. `sprint-status.yaml` and whatever else BMAD reads are projections regenerated from the tracker, never the master copy. How that is materialised is a design decision for the architecture. |
| Platforms | **GitLab and GitHub, both in version 1**, with full story-cycle parity. The tracker and the git remote may be different platforms (code on GitLab, issues on GitHub, and vice versa). GitLab CE must work: no reliance on Premium-only epics, blocking issue links or merge trains. |
| Scope of v1 | The complete story cycle: setup in a consumer project, initial synchronisation of PRD/epics/stories, taking a story, isolated worktree and branch, merge request, CI gate on the pushed commit, closing the story, projection back into BMAD's files, and a way to know what to work on next. **`bmad-loop` support is after v1**; v1 must not make it impossible (see the `ci-status.json` contract in §5). |
| Dogfooding | BMAD specifies, builds and tests this plugin. **The plugin tracks its own stories in this repository's issues as soon as an increment exists that can do it.** The planning must produce a first increment that makes that possible early. |
| Test bed | A GitLab project on an internal host is available for real-platform tests. **Its host name never appears in this repository, in commits, issues or pull requests**: it is read from local configuration or the environment and referred to as "the lab GitLab host". |
| Language | English for code, documents, commits and issues. |

## 3. Verified facts about the environment

### 3.1 BMM 6.12.0 (read from the `v6.12.0` tag of `bmad-code-org/BMAD-METHOD`)

- Install: `npx bmad-method@6.12.0 install --directory . --yes --modules bmm --tools claude-code
  [--set <module>.<key>=<value>] [--custom-source <path|git-url>]`. `--tools` is required with
  `--yes`. **Shims are off on a fresh install**: `bmad-create-prd`, `bmad-create-story`,
  `bmad-dev-story`, `bmad-sprint-status`, `bmad-generate-project-context` are not installed unless
  `--shims`. The skills that exist by default are the `plan/` and `build/` families:
  `bmad-project-context`, `bmad-product-brief`, `bmad-prd`, `bmad-architecture`, `bmad-ux`,
  `bmad-create-epics-and-stories`, `bmad-sprint-planning`, `bmad-spec`, `bmad-build`,
  `bmad-build-auto`.
- What the installer writes and what is meant to be committed: `_bmad/config.toml` (team answers,
  regenerated each install: commit), `_bmad/custom/` (human-authored `<skill>.toml` overrides, never
  touched by the installer: commit; `*.user.toml` ignored), `_bmad/render/` (rendered skill
  snapshots: ignored), `_bmad/scripts/` (`resolve_customization.py`, `resolve_config.py`,
  `memlog.py`, `render_skill.py`, wiped each install), `_bmad/_config/` (manifests), per-module
  `config.yaml` (generated), `.claude/skills/<skill>/` (the skills themselves, copied by the
  installer). Default paths: `output_folder = _bmad-output`, `planning_artifacts =
  {output_folder}/planning-artifacts`, `implementation_artifacts =
  {output_folder}/implementation-artifacts`, `project_knowledge = docs`.
- **Two config-reading styles coexist.** `bmad-product-brief`, `bmad-prd`, `bmad-ux`,
  `bmad-create-epics-and-stories`, `bmad-sprint-planning` read `_bmad/bmm/config.yaml`;
  `bmad-architecture`, `bmad-spec`, `bmad-project-context` read the four-layer TOML merge through
  `resolve_config.py`; `bmad-build*` bake TOML values into their rendered snapshot. A path pinned in
  `_bmad/custom/config.toml` is invisible to the first group. **Paths are set at install with `--set`.**
- Artefact paths: PRD run folder `{planning_artifacts}/prds/prd-{project_name}-{date}/prd.md`
  (`status: draft` → `final`, plus `addendum.md`, `.memlog.md`); several PRD folders can coexist,
  and nothing selects among them automatically. Architecture
  `{planning_artifacts}/architecture/architecture-{project_name}-{date}/ARCHITECTURE-SPINE.md`.
  Epics: one fixed `{planning_artifacts}/epics.md` (`## Epic N: Title` / `### Story N.M: Title`).
  Sprint status: **one** `{implementation_artifacts}/sprint-status.yaml` per project, generated by
  `sprint_plan.py` (keys `epic-N`, `N-M-kebab-title`, `epic-N-retrospective`; story statuses
  `backlog | ready-for-dev | in-progress | review | done`; never downgrades). Story specs:
  `{implementation_artifacts}/spec-{key}.md` with frontmatter `status: draft → ready-for-dev →
  in-progress → in-review → done` (`blocked` in build-auto), sections `Intent`, `Boundaries`,
  `I/O & Edge-Case Matrix`, `Code Map`, `Tasks & Acceptance`, `## Review Triage Log`, and in
  build-auto an `## Auto Run Result`. Epic-sized alternative: `bmad-spec` writes
  `{output_folder}/specs/spec-{slug}/SPEC.md` + `stories.yaml` (no sprint-status involved).
- **Known gaps in 6.12.0.** The discovery globs of `bmad-create-epics-and-stories`
  (`*prd*.md`, `*architecture*.md`) do not see the new `prds/` and `architecture/` run folders: the
  paths have to be given explicitly. `sprint_plan.py` detects a story file as `{key}.md`, not
  `spec-{key}.md`. `bmad-build` marks the spec `done` but writes only `review` into
  `sprint-status.yaml`; `bmad-build-auto` never touches `sprint-status.yaml`. Neither build skill
  picks "the next story" by itself.
- Hooks: `_bmad/custom/<skill>.toml` merges over the skill's `customize.toml`; scalars override,
  arrays append, `[[…]]` arrays merge by `id`/`code`; `<skill>.user.toml` wins over `<skill>.toml`.
  Keys: `activation_steps_prepend`, `activation_steps_append`, `persistent_facts`, `on_complete`
  (string or array), `implementation_handoff`, review layers. `bmad-build` fires `on_complete` at
  the end of its present step; **`bmad-build-auto` fires it on every terminal status, including
  `blocked`**. `resolve_customization.py --skill <dir> --project-root <root> --key workflow` returns
  the merged result as JSON (Python ≥ 3.11).
- **The installer does not place a module's Python anywhere runnable.** A custom module's skills
  are copied to `.claude/skills/<skill>/` (their `scripts/` included); `_bmad/scripts/` is core-only
  and wiped on each install. A hook that needs the module's CLI has to reach it through `uv`
  (`uv run --with <package>==<version> …` or `uvx --from git+…@<tag> …`) or through a file the
  module's own setup step deployed to a stable path.
- Custom module packaging for the classic installer: Discovery mode needs
  `.claude-plugin/marketplace.json` (`plugins[]` with `name`, `source`, `version`, `skills[]`) plus
  `module.yaml` + `module-help.csv`; Direct mode scans for `SKILL.md` directories. Only the listed
  skill directories are copied.
- Headless (non-interactive) support: `bmad-prd`, `bmad-architecture`, `bmad-ux` have
  `references/headless.md` (JSON status contract: `complete | partial | blocked`, artefact paths);
  `bmad-product-brief`, `bmad-sprint-planning`, `bmad-spec` have inline headless sections;
  `bmad-build-auto` is unattended by nature; **no headless path**: `bmad-create-epics-and-stories`,
  `bmad-build`, `bmad-project-context`.

### 3.2 GitLab and GitHub CLIs (all verified against real projects)

- `gh api --paginate` and `glab api --paginate` concatenate pages with **no separator**; parse with
  a JSON `raw_decode` loop.
- `gh api` **switches to POST as soon as a `-f`/`-F` field is given**: a listing needs `-X GET`.
  `glab api` does the same and needs `--method GET`.
- `gh api` has **no `-R` flag**: the repository is part of the path (`repos/{owner}/{repo}/…`).
  `gh pr`, `gh issue`, `glab mr`, `glab label` take `-R`; `glab api` takes `--hostname`.
- GitHub `pulls?head=` requires **`owner:branch`**; a bare branch name makes the API ignore the filter
  and return **every open PR**. `gh pr close --delete-branch` is a boolean flag (no `false` value).
- `glab api -F` takes **one `key=value` token** per flag; the project path in `projects/<path>/…`
  must be **URL-encoded** (`group%2Frepo`).
- A raw space in a URL is a silent `[]` on gh and an HTTP 400 on glab: pass query text as fields.
- GitHub's search index lags a create by ~5 s: look a freshly created issue up by REST list, not by
  search. `gh issue create` prints a URL, not JSON. `gh pr merge` prints nothing on stdout.
  `gh issue edit --add-label` fails on an unknown label; GitLab creates unknown labels.
- `gh run list --branch … --commit <sha>` filters server-side and an **empty `--commit ""` is no
  filter at all**; the flag needs gh ≥ 2.21. GitLab has no equivalent on the MR-pipelines endpoint:
  filter the listing on each entry's `sha`. Do not switch to `projects/…/pipelines?sha=`: it loses
  the per-MR scoping.
- Label separators: `::` on GitLab (scoped labels), `:` on GitHub. GitLab issue updates through the
  API replace the whole label list; GitHub `--add-label/--remove-label` preserve the rest.
- Neither API offers a conditional write on issues: two concurrent "take this story" writes can both
  succeed. Read-after-write is the only detection available.
- gh's token may live in a system keyring; inside a sandbox (`ai-jail`, containers) it is not
  reachable and must be passed as `GH_TOKEN`. SSH keys with passphrases are unusable in
  non-interactive shells; HTTPS remotes with `gh auth git-credential` work everywhere.

### 3.3 The agent's execution environment

- Claude Code's Bash tool times out at **120 s by default, 600 s at most**, and refuses a foreground
  `sleep N && …`. Anything the coding agent runs (hooks included) must finish inside that, so long
  waits (a CI pipeline) must be resumable across calls.
- A hook runs inside the HOST skill's session: the only things it can rely on are the project root,
  `uv`, git, and the CLIs the user has authenticated.
- BMAD's own coding-agent variability is real: the same instruction text was executed differently
  by different sessions in the predecessor's lab. The less the hook leaves to interpretation, the
  better.

## 4. Lessons from the predecessor (44 defects, all confirmed against real repositories)

Grouped by what the design must make impossible, not by history.

- **Reading the wrong file.** The predecessor read `{planning_artifacts}/prd.md`; BMM writes the
  PRD in a per-run folder. It also invented spec paths (`{implementation_artifacts}/{key}.md`)
  instead of resolving them once. → Every path BMAD writes is resolved by one shared routine,
  driven by the facts in §3.1, and covered by a test with the real layout.
- **A gate that reads nothing and answers green.** Twice: the CI gate ran before the merge
  request existed (`no MR` → green), and once the MR existed, the lookup read the previous
  commit's run for the first seconds after a push. → The gate reads the pipeline of the exact
  pushed commit, treats "nothing found yet" as *running*, never as green, and unknown pipeline
  states as running, never as green. `skipped`/`neutral` are green; `manual`, `action_required`,
  `canceled`, `timed_out` are red (a pipeline parked on an operator never goes green by itself).
- **Two names for one thing.** Story issues titled from the spec and from the key drifted into
  `Story 1.1: Story 1.1: Login Form` on one path and `Story 1.1: Login Form` on another, and the
  title is what deduplication keys on; the MR title used the key while the issue used the title.
  → One title routine, one format per artefact, MR and issue share it. The predecessor's formats
  are known to work: `PRD: {prd_key}`, `Epic {n}: {title}`, `Story {epic}.{story}: {title}`,
  `Retrospective: Epic {n}`.
- **State that lived in no commit.** Hooks updated the tracker from a file that was never
  committed, and the next checkout threw the change away; other hooks staged the whole worktree and
  swept unrelated files into the commit. → Whatever is mirrored is committed; whatever is committed
  is named explicitly; generated files (`ci-status.json`) are ignored, never committed.
- **A plugin that never closed anything.** The post-merge helper listed pull requests with a flag
  the CLI does not have, treated the failure as "nothing to do" and exited 0 for months. → Every
  call to gh/glab is exercised against a real project before merge; a failed lookup is a failure,
  not an empty result.
- **Cross-platform blindness.** Steps annotated with the *tracker* platform were skipped when the
  *git* platform differed, so MR and CI steps silently did nothing. → Tracker and git remote are
  two explicit coordinate sets, chosen independently, tested in both mixed configurations.
- **Long waits inside a single call.** A 30-minute poll in one command was killed by the tool
  timeout, so the verdict was never written. → No single invocation blocks longer than ~100 s;
  waiting is resumable.
- **Concurrency by accident.** `bmad-loop` squashes the story worktree into one commit and discards
  the hook's pushed commits; a story worktree inherited stale generated files; two runs of the same
  story left orphan worktrees and branches. → The design must state what happens when the same
  story is taken twice, when a worktree is stale, and when an external runner rewrites history.
- **Untestable logic.** 1,354 tests could only lint the YAML; every real defect needed a live LLM run
  to surface. → Logic lives where pytest can run it without an LLM; platform calls are tested
  against real projects; the agent's role is reduced to text.

The full list (D01–D44, S1–S11, R1–R24) with evidence is in the predecessor repository's issues:
https://github.com/pasqualtroncone/bmad-issue-tracking/issues?q=is%3Aissue (all closed) and its
`CLAUDE.md` on branch `devel`.

## 5. Contracts inherited from the ecosystem (facts, to honour or consciously break)

- **`bmad-loop` `[verify]`**: a script exit 0 = CI green, 1 = red or unknown, reading a
  `ci-status.json` at the story worktree root with `{"status": "green" | "red", …}`. bmad-loop is
  the single writer of `sprint-status.yaml` in its flow, merges stories locally, never pushes, and
  fires a `post_merge` plugin hook with `BMAD_LOOP_BRANCH`, `BMAD_LOOP_STORY_KEY`,
  `BMAD_LOOP_RUN_DIR`, `BMAD_LOOP_REPO_ROOT` in the environment. v1 does not implement bmad-loop
  support but must not preclude it.
- **BMM story keys** (`N-M-kebab-title`) and the sprint-status vocabulary (§3.1) are what BMAD's
  own scripts and skills understand; a projection that BMAD reads must speak them exactly.
- **BMAD's classic installer** is the only released install route (6.12.x); the Skills-as-modules
  route exists only on BMAD `main`.

## 6. What the wider ecosystem does (reference reading, not requirements)

Researched 2026-09-22: **ccpm** (GitHub Issues as project state, one worktree per issue, parallel
agents, GitHub-only, bash scripts for tracking "with no LLM overhead"); **Beads / Gas Town** (a graph
issue tracker built for concurrent agents, `ready`/claim semantics, its own database rather than a
GitLab/GitHub mirror); **Vibe Kanban**, **Symphony** (kanban/Linear → isolated agent runs);
**Spec Kit**, **OpenSpec**, **Superpowers** (specification and discipline, no tracker). None offers
BMAD's planning depth together with multi-writer parallelism; the "tracker as shared state" idea is
the one that addresses our problem.

## 7. Process requirements for the planning and the build

- The planning artefacts (`brief`, `prd.md`, `ARCHITECTURE-SPINE.md`, `epics.md`,
  `sprint-status.yaml`, story specs) are **committed** in this repository under `_bmad-output/`.
  They are the specification and the proof of the method.
- The product owner approves the PRD and the architecture before epics are cut.
- The first increment that can install itself in this repository and track its own stories comes
  **early**, so that the rest of the build is dogfooded; how early is the planning's call.
- Every change arrives by merge request with its tests; anything touching GitLab or GitHub is
  exercised against a real project of that platform before merge; a check that the lab GitLab host
  name is absent from the repository runs on every merge request.
- Predecessor material is reference only: read `../bmad-issue-tracking/CLAUDE.md` (branch `devel`)
  for the hook model, title formats, platform differences and CI state rules; never copy its code.
- Questions during planning or build go to the product owner, or to the reference Claude session
  named in `CLAUDE.md`, which holds the evidence behind every fact in this brief.
