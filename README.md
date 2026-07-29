# Plan-Driven Workflow

A Claude Code plugin that drops a **plan-driven development workflow** into any repository with one command.

It provides the `/init-my-repo` skill, which auto-detects your repo's conventions, interviews you to
confirm them, writes a single `.claude/docs-skills.md` config, and installs a small chain of
config-driven workflow skills:

| Skill | What it does |
|-------|--------------|
| `design-doc` | Reason out **what** to build and **why**; maintains a design corpus |
| `plan-feature` | Turn a design (or a discussion) into a phased implementation plan — and split it into milestone plans when it outgrows one session |
| `execute-plan` | Implement a plan (or a single milestone) wave-by-wave, then open a PR. Should be run with Opus models, spawns its own Sonnet agents. |
| `execute-umbrella` | Run a multi-milestone umbrella plan: wave the milestones, worktree and branch each one, dispatch an orchestrator per milestone, merge as they go green. Should be run with Opus models. Spawns /execute-plan for each milestone. |
| `audit-plan` | Verify a plan was implemented (and conforms to the design), then archive it |

The installed skills are **generic and identical in every repo** — everything project-specific lives
in `.claude/docs-skills.md`, so you re-tune a repo by editing one config file and upgrade the skill
logic in one place.

## Install

In Claude Code, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add DawidIzydor/DesignPlanSkills
/plugin install plan-workflow@plan-workflow
```

> You can also point at a full Git URL or a local path:
> `/plugin marketplace add https://github.com/DawidIzydor/DesignPlanSkills.git`
> `/plugin marketplace add ./plan-workflow` (for local testing)

Once installed, the `/init-my-repo` command is available in every session.

## Usage

From inside the repository you want to set up:

```
/init-my-repo
```

It will:

1. **Detect** your conventions (test runner, docs layout, main branch, PR tool, shell).
2. **Run the built-in `/init`** first if there's no `CLAUDE.md`, then harvest answers from it.
3. **Interview** you to confirm the detected defaults — spending its attention on your repo's
   cross-cutting invariants (the rules every implementing agent must always hold).
4. **Write** `.claude/docs-skills.md`.
5. **Install** the workflow skills into the repo's `.claude/skills/` (always `plan-feature`,
   `execute-plan`, `execute-umbrella`, `audit-plan`; plus `design-doc` if you opt into a design
   corpus).

Then drive the workflow:

```
/design-doc         # (if installed) reason out what to build and why
/plan-feature       # turn a design or discussion into a phased plan (splits into milestones if it's large)
/execute-plan       # implement a plan, or a single milestone, wave-by-wave, then open a PR
/execute-umbrella   # run a whole multi-milestone family, one worktree/branch/PR per milestone
/audit-plan         # verify + archive a completed plan
```

## How it's packaged

This repo is **both a plugin and a self-hosting marketplace**:

- `.claude-plugin/plugin.json` — the plugin manifest (name, version, points `skills` at `./skills/`).
- `.claude-plugin/marketplace.json` — a one-plugin marketplace whose source is `./` (this repo).
- `skills/init-my-repo/` — the entry-point skill. Its `assets/skills/` carries the five workflow
  skills that `/init-my-repo` installs into target repos, and `assets/docs-skills.template.md` +
  `references/config-schema.md` back the config it writes.

The five workflow skills are intentionally **not** exposed at the plugin level — they're installed
per-repo by `/init-my-repo` so each project gets its own copy wired to its own config.

## Config reference

The full list of `.claude/docs-skills.md` keys, defaults, and per-test-runner detection lives in
[`skills/init-my-repo/references/config-schema.md`](skills/init-my-repo/references/config-schema.md).

## License

MIT — see [LICENSE](LICENSE).
