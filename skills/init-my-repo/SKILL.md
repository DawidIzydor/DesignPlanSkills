---
name: init-my-repo
description: >
  Bootstraps the plan-driven workflow (design-doc, plan-feature, execute-plan, execute-umbrella,
  audit-plan) into the current repository. Auto-detects the repo's conventions, interviews the user to
  confirm them, writes a single `.claude/docs-skills.md` config, installs the generic workflow skills
  into `.claude/skills/`, and runs the built-in `/init` first when the repo has no CLAUDE.md. Use this
  whenever the user wants to set up, initialize, bootstrap, or migrate the planning workflow into a
  project — phrases like "init my repo", "set up the plan skills here", "migrate design-doc /
  plan-feature / execute-plan / execute-umbrella / audit-plan into this project", "add the planning
  workflow to this repo", "bootstrap this repo for plans", or "/init-my-repo". Trigger even when the
  user names only one of the skills but clearly wants it working in a different project than the one it
  came from.
---

## What this skill does

Turns a bare repository into one where the plan-driven workflow just works. In one pass it:

1. **Detects** the repo's conventions (test runner, docs layout, main branch, PR tool, shell).
2. **Runs `/init`** first if there's no `CLAUDE.md`, then harvests answers from it.
3. **Interviews** the user to confirm the detected defaults — the one thing it can't guess is the
   repo's cross-cutting invariants, so it works hardest there.
4. **Writes** `.claude/docs-skills.md` — the single per-repo config every workflow skill reads.
5. **Installs** the generic workflow skills into the repo's `.claude/skills/`: always
   `plan-feature`, `execute-plan`, `execute-umbrella`, and `audit-plan`; plus `design-doc` when the
   repo opts into an upstream **design corpus** (the design → plan → code chain).

The design is **config-driven**: the installed skills are identical in every repo and stay generic.
Everything project-specific lives in `.claude/docs-skills.md`. That means you re-tune a repo by
editing one config file, and you upgrade the skill logic in one place — the skills never drift
per-repo. This mirrors how `plan-feature` already reads its paths from that file.

**The design corpus is opt-in.** Some repos reason out *what to build and why* in design docs before
planning the *how*; many don't. The `design-docs-dir` config key is the switch: set it and the four
skills form a full chain (`design-doc` maintains the corpus, `plan-feature` plans from it, `audit-plan`
checks the code back against it); leave it `none` and `plan-feature`/`audit-plan` run exactly as they
did before the chain existed, and `design-doc` isn't installed. Confirm which the repo wants in the
interview (Step 3).

> **Why not bake the specifics into each skill copy?** Because then every repo's skills drift
> independently and re-tuning means regenerating prose. One config file the skills read is DRY,
> reviewable in a diff, and re-runnable.

---

## Step 0 — Confirm the target

This skill acts on the **current working directory's repository**. Before doing anything:

- Confirm you are inside a git repo: `git rev-parse --show-toplevel`. If not, tell the user and
  offer to `git init` — the skills reference branches and PRs, so a repo is required.
- If `.claude/docs-skills.md` **already exists**, this is a re-run or a partially-configured repo.
  Read it, treat its values as the current defaults, and **merge** rather than clobber (Step 4).
  Tell the user you found an existing config and will update it.

The bundled skills and template this skill installs live under **this skill's own base directory**
(the path reported when the skill loaded), in `assets/`. Resolve that base dir once now; every copy
in Step 5 reads from `<base>/assets/`.

---

## Step 1 — Detect the repo's conventions

Run these detections in parallel and hold the results as *proposed defaults* — you'll confirm them
with the user in Step 3, never silently commit to them.

**Version control**
- Main branch: inspect `git symbolic-ref refs/remotes/origin/HEAD` (strip to the branch name); if
  that fails, check whether `main` or `master` exists locally. Default `main`.
- PR tool: is `gh` on PATH (`gh --version`)? Then `glab`? If neither, `pr-tool: none`.

**Shell / OS**
- On Windows default `shell: pwsh`; on macOS/Linux default `shell: bash`. This only decides which
  syntax the installed skills use for their few filesystem commands — git commands are identical
  everywhere.

**Test runner** — detect from manifest files (read the actual test script when one exists):

| Found in repo | Proposed `test-command` |
|---|---|
| `package.json` (read `scripts.test`) | `npm test` (or the script's real command) |
| `*.csproj` / `*.sln` | `dotnet test` |
| `Cargo.toml` | `cargo test` |
| `go.mod` | `go test ./...` |
| `pyproject.toml` / `pytest.ini` / `setup.py` | `pytest` |
| `pom.xml` | `mvn test` |
| `build.gradle` | `./gradlew test` |

Also propose a `test-filter-template` for the same runner (see `references/config-schema.md` for
the per-runner filter syntax) and guess `tests-are-slow` — true for compiled/integration-heavy
suites, false for fast unit suites.

**Docs layout** — glob for existing directories, matching the repo's own casing:
- Plans: `docs/plans/`, `Docs/plans/`, `doc/plans/` → else default `docs/plans/`
- Backlog: `docs/backlog/` → else default `docs/backlog/`
- Architecture docs: `docs/architecture/`, `Docs/architecture/`, `docs/adr/` → else `docs/architecture/`
- Audited-plans archive: `docs/old_plans/audited/` → else default `<plans-dir>/../old_plans/audited/`
- Design corpus: `docs/design/`, `Docs/design/`, `docs/spec/`, `docs/rfcs/` → if one exists (or it
  contains a `00-vision.md`), propose it as `design-docs-dir` and plan to install `design-doc`. If none
  exists, propose `design-docs-dir: none` and *offer* the design chain in the interview — don't force
  it on a repo that doesn't work that way.

**Companion skills**
- Is there an `update-arch-docs` skill (glob `.claude/skills/update-arch-docs/` and
  `~/.claude/skills/update-arch-docs/`)? Sets `has-update-arch-docs-skill`.

---

## Step 2 — CLAUDE.md via `/init`

Check for `CLAUDE.md` (or `.claude/CLAUDE.md`) at the repo root.

- **Missing:** run the built-in `/init` now, via the `init` Skill. It analyzes the codebase and
  writes `CLAUDE.md`. Do this *before* the interview, because the generated file is a goldmine for
  the two things hardest to auto-detect: the **build/test commands** and the **cross-cutting
  invariants** (often under an "Architecture", "Conventions", or "Non-negotiable rules" heading).
- **Present:** read it. Pull candidate test commands and invariants from it to pre-fill Step 3.

Either way, after this step you should have a draft list of invariants harvested from `CLAUDE.md`
rather than starting the interview from nothing.

---

## Step 3 — Interview: confirm the defaults

Show the user a compact table of everything detected — proposed value per key — and ask them to
confirm or correct. Lead with what you're most and least sure about.

Auto-detected mechanical settings (paths, test command, main branch, PR tool, shell) are usually
right; present them for a quick yes/no rather than interrogating each one.

**Spend your attention on the cross-cutting invariants.** These are the rules every implementing
agent must hold at all times — threading constraints, "config lives in JSON not code", a SOLID
doc to follow, "delete dead code as you go", naming conventions. They are the highest-value part
of the config (they get injected verbatim into every phase agent's prompt in `execute-plan`) and
the part you cannot reliably guess. Propose the candidates you harvested from `CLAUDE.md`, then
ask the user directly: *"What must an agent never do, and what must it always do, when changing
code here?"* Write down what they say.

**Settle the design-corpus question.** This one decision changes what gets installed, so ask it
directly: *"Do you reason out what to build and why in design docs before planning how to build it —
and want plans checked back against those docs?"* If the repo already has a `docs/design/` corpus
(you'd have found it in Step 1), lead with that as a yes. On **yes**: set `design-docs-dir` to the
corpus path (default `docs/design/`) and plan to install `design-doc`. On **no**: set
`design-docs-dir: none` and skip `design-doc` — `plan-feature` and `audit-plan` will run in their
classic, design-agnostic mode. Don't oversell it; plenty of good repos don't work this way.

Use `AskUserQuestion` for the handful of genuine either/or decisions (e.g. the design-corpus opt-in
above, `pr-tool` when both `gh` and `glab` are present, or an ambiguous main branch). Don't turn
confirmable defaults into a questionnaire.

---

## Step 4 — Write `.claude/docs-skills.md`

Write the config from `<base>/assets/docs-skills.template.md`, substituting the confirmed values.
Keep the template's format — a **Paths** table, a **Workflow** table, a **Testing** table, and a
prose **Cross-cutting invariants** section — because that's the shape the installed skills parse
and the existing convention in repos that already have this file.

**If a config file already existed** (Step 0): merge. Preserve any keys already present (another
skill may rely on them — e.g. `ui-docs-dir`, `arch-template`), update the values the user changed,
and add the new keys. Never drop a key you don't recognize.

See `references/config-schema.md` for the authoritative key list, defaults, and per-runner filter
syntax.

---

## Step 5 — Install the skills

Copy each bundled skill directory from `<base>/assets/skills/` into the target repo's
`.claude/skills/`. Always install these four:

- `<base>/assets/skills/plan-feature/`      → `<repo>/.claude/skills/plan-feature/`
- `<base>/assets/skills/execute-plan/`      → `<repo>/.claude/skills/execute-plan/`
- `<base>/assets/skills/execute-umbrella/`  → `<repo>/.claude/skills/execute-umbrella/`
- `<base>/assets/skills/audit-plan/`        → `<repo>/.claude/skills/audit-plan/`

`execute-umbrella` is not optional even in a repo that has no umbrella plans yet: `execute-plan`
hands off to it by name whenever it's pointed at one, and `plan-feature` produces umbrellas on its
own whenever a plan outgrows a session. Installing only three leaves a dangling reference that
surfaces at the worst moment.

Install `design-doc` **only if the repo opted into the design corpus** (Step 3 — `design-docs-dir` is
a real path, not `none`):

- `<base>/assets/skills/design-doc/`    → `<repo>/.claude/skills/design-doc/`

Copy them **verbatim** — they are fully generic and read everything repo-specific from the config
you just wrote. There is no per-repo substitution to do inside them. (A repo may later specialize its
`design-doc` copy — e.g. tune the vocabulary to its domain — but install the generic one first.)

**Don't clobber silently.** If a skill of the same name already exists in the target repo, stop and
show the user the difference (is it their own customized version, or an older copy from this
skill?). Overwrite only with their say-so; otherwise install under a suffixed name or skip it and
report.

---

## Step 6 — Report and orient

Tell the user, concisely:

- What was written: the config path, and the skills installed (or skipped) — note whether
  `design-doc` and the design chain were turned on.
- Whether `/init` ran and produced a `CLAUDE.md`.
- Any judgment calls you made on their behalf (a guessed test command, an inferred invariant) so
  they can correct them.
- How to drive the workflow from here:
  - `/design-doc` — *(if installed)* reason out what to build and why; produces design-locked docs
  - `/plan-feature` — turn a design-locked system (or a discussion) into a phased plan under `plans-dir`
  - `/execute-plan` — implement a plan wave-by-wave, then open a PR
  - `/audit-plan` — verify a plan was implemented and (with a design corpus) conforms to the design,
    then archive it

Offer a smoke test: *"Want me to write a tiny throwaway plan and run `/audit-plan` on it to confirm
the wiring end-to-end?"* — cheap confidence that the config paths resolve.

---

## Edge cases

- **Not a git repo:** offer `git init`; don't proceed silently — branch/PR steps need it.
- **Config already present:** merge, preserve unknown keys, report what changed (Step 0 / Step 4).
- **A target skill already exists:** show the diff, ask before overwriting (Step 5).
- **No test runner detected:** ask the user for the test command outright; leave `test-command`
  blank only if they truly have no suite, and note that `execute-plan`'s test phase will then ask
  at run time.
- **No `update-arch-docs` skill in the target repo:** set `has-update-arch-docs-skill: false`;
  `execute-plan` will update docs in `arch-docs-dir` directly instead of delegating.
