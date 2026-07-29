---
name: execute-plan
description: >
  Executes a feature plan from the project's plans directory by reading its phase structure,
  determining which phases can run in parallel based on their "Depends on:" declarations, and
  spawning agents to implement them. Independent phases in the same wave run concurrently in the
  current working directory (this skill assumes it is already running inside a dedicated branch or
  worktree, so it does not create further worktrees); dependent phases run in order. After all
  phases complete, updates architecture documentation and creates a PR. Invoke this skill whenever
  the user says "execute plan", "run plan", "implement plan [name]", "do the plan", "start the
  plan", "run all phases of X", "implement the whole X plan", or points to a plan file and asks to
  implement or run it — even if they just say "let's do this plan" while a plan file is in context.
---

## What this skill does

1. Reads a plan from the configured `plans-dir`, extracts phases and their dependencies
2. Groups unfinished phases into **execution waves** (phases in the same wave are independent and safe to run in parallel)
3. Runs each wave — each phase writes its core tests **first**, from the phase spec, then implements against them; phases in a multi-phase wave run concurrently in the current working directory
4. Commits the wave's work after every wave completes
5. Runs the full suite once and audits that the phase tests still assert what they were written to assert
6. Updates architecture documentation
7. Creates a PR with a summary of all implemented phases

> **Assumption — no nested worktrees.** This skill is meant to be launched from inside a worktree (or at least a dedicated feature branch). It deliberately does **not** spawn its own worktrees for parallel phases, because nesting a worktree inside a worktree buys no extra isolation and only adds merge bookkeeping. Parallel phases stay safe instead because, by design, independent phases touch disjoint files — see Step 3.

---

## Step 0 — Read project config

Read `.claude/docs-skills.md` in the project root. It carries this repo's paths, workflow, test
command, and cross-cutting invariants. Extract the keys this skill uses; if the file is missing,
fall back to the defaults noted and tell the user no config was found.

| Key | Used for | Default if absent |
|-----|----------|-------------------|
| `plans-dir` | where plans live | `docs/plans/` |
| `arch-docs-dir` | docs to update in Step 5 | `docs/architecture/` |
| `main-branch` | the branch never to commit to directly | `main` |
| `branch-prefix` | prefix for a new feature branch | `plan/` |
| `pr-tool` | how Step 6 opens a PR — `gh` / `glab` / `none` | `gh` if on PATH, else `none` |
| `worker-model` | model for spawned phase agents | `sonnet` |
| `has-update-arch-docs-skill` | delegate docs to that skill vs. do it inline | `false` |
| `test-command` | the full-suite test command (Step 4) | ask the user |
| `test-filter-template` | run a subset by name; `{NAME}` is the placeholder | none (skip narrowing) |
| `tests-dir` | where each phase writes its tests (Step 3) | infer from the repo layout |
| `tests-are-slow` | whether to run-twice-and-narrow or just run fully | infer from suite size |
| `shell` | `bash` or `pwsh` for filesystem commands | detect from OS |
| `invariants` | the "Cross-cutting invariants" section, injected into every phase agent | none |

Where a command below is shown in bash, translate to the `shell` value if it's `pwsh` (e.g.
`rm -rf` → `Remove-Item -Recurse -Force`). Git and `gh`/`glab` commands are identical in both shells.

---

## Step 0.5 — Check whether this is an umbrella, not a plan

Locate the plan first: if the user gave a path, use it; if they named a plan, search `plans-dir`.
Then check what kind of file it is before doing anything else, because one kind can't be run here.

A **milestone umbrella** declares **no phases of its own** — no `## Phase N` sections, no per-phase
status table — and instead carries a **Milestones table** linking out to sibling plan files (`M1`,
`M2`, …), usually under a banner saying it is not executable. `plan-feature` produces one whenever a
plan outgrows a single session.

If that's what you're holding, **stop and run the `execute-umbrella` skill on it instead** — read
`.claude/skills/execute-umbrella/SKILL.md` and follow it. It schedules the milestones into waves,
gives each its own worktree, branch and PR, and dispatches an agent per milestone that comes back
here to run *this* skill against one milestone file. Continuing here would find no phases and report
that there is nothing to implement — true of the file, false of the work.

Everything else — an ordinary plan, or a single milestone file (it has phases *and* a `Runs after:`
header) — is this skill's job. Carry on.

---

## Step 1 — Read the plan

If the user gave you a path, read that file. If they referred to a plan by name, search `plans-dir` for a match and read it.

Extract:
- **Plan title** (H1 heading) — used for branch/PR naming
- **Phase list** — for each phase: its number, title, "Depends on:" declaration, and current status from the Status table
- **"After implementation"** section — which architecture docs need updating
- **Cross-cutting invariants** section — note it, but the authoritative copy is the `invariants` from `.claude/docs-skills.md` (Step 0). If the plan restates them, prefer the config's copy when they differ.

Skip any phase already marked ✅ Done in the status table.

---

## Step 2 — Build the execution plan

### Determine dependencies

For each unfinished phase, read its `Depends on:` line:
- "Depends on: nothing" or absent → **independent** (no predecessor)
- "Depends on: Phase N" → must run after Phase N
- "Depends on: Phases N and M" → must run after both N and M

### Build execution waves

Assign each phase to the earliest wave where all its dependencies have been assigned to earlier waves.

Example:
- Wave 0: Phase 1 (independent), Phase 2 (independent) → **run in parallel**
- Wave 1: Phase 3 (depends on Phase 2)
- Wave 2: Phase 4 (depends on Phase 3)
- Wave 3: Phase 5 (depends on Phases 2–4)

**Present the wave breakdown to the user before starting.** Show a table like:

```
Wave 0 [parallel]:   Phase 1 — <title> | Phase 2 — <title>
Wave 1 [sequential]: Phase 3 — <title>
Wave 2 [sequential]: Phase 4 — <title>
Wave 3 [sequential]: Phase 5 — <title>
```

Let the user confirm or correct it. Start only after confirmation.

### Make sure you are on a dedicated branch

All work must land on a branch, never directly on `main-branch`. Check the current branch first:

```bash
git rev-parse --abbrev-ref HEAD
```

- If you are already on a dedicated branch (the common case — you were launched inside a worktree), stay on it. Do **not** create another branch or worktree.
- If you are on `main-branch`, create a feature branch so the work is isolated:

```bash
git checkout -b <branch-prefix><kebab-case-plan-title>
```

---

### Worker model

**Every agent you spawn in this skill launches with `model: "<worker-model>"`** (from Step 0; default `sonnet`). This is independent of whatever model the orchestrator (this session) runs on. The reasoning: orchestration — wave/dependency analysis, merge handling, deciding what to do on failure — is the judgment-heavy part and stays on the session's model; per-phase implementation is mechanical fan-out work a cheaper, faster model handles well.

Pass `model: "<worker-model>"` explicitly on **every** `Agent` call below — phase agents and any docs agent. Don't omit it and rely on inheritance, which would pull in the orchestrator's model. (If `worker-model` is `inherit`, omit the override and let agents inherit the session model.)

## Step 3 — Execute waves in order

Process each wave sequentially. Within a wave, phases may run in parallel.

**Each phase writes its own core tests first, from the phase spec, before it writes production code** — see the Phase Agent Prompt. The reason is provenance: a test written afterwards, by reading the finished implementation, can only restate what that code already does, bugs included. It cannot tell you the phase built the *wrong thing*. A test written from the phase spec and its `Implements (design):` rule encodes the intent, so an implementation that drifts from the plan shows up as a failure instead of being quietly blessed.

Keep the runs narrow. **Nobody runs the full suite until Step 4** — a phase runs only its own tests, via `test-filter-template`, and only when it is alone in its wave (concurrent builds in one working directory collide on shared build output).

No phase agent runs in its own worktree. Every agent works directly in the current working directory, and **agents never run git themselves** — you (the orchestrator) own all commits. This keeps the git index race-free even when several agents edit files at the same time.

### Single-phase wave

Spawn one Agent (no isolation, `model: "<worker-model>"`) with the Phase Agent Prompt below, filling
the `<wave-mode>` block with the **alone-in-the-wave** variant — this agent may run its own tests, so
it gets the full red-then-green cycle.

Wait for it to complete, then commit:

```bash
git add -A && git commit -m "Phase <N>: <phase title>"
```

### Multi-phase wave (2+ phases)

Spawn one Agent **per phase**, all with **no isolation** and `model: "<worker-model>"`, in a single message so they run concurrently in the current working directory. Fill each `<wave-mode>` block with the **sharing-the-wave** variant.

This is safe because independent phases in the same wave touch disjoint files by design — that is exactly what made them eligible to share a wave. Because the agents do not commit, there is no branch to merge afterward; their edits simply accumulate in the working tree.

**Run the wave's tests yourself once the agents are done.** Agents sharing a wave can't run the build — concurrent builds collide on shared output — so they write their tests first and implement, but never see them go green. You are the first point where that can safely happen. Run just this wave's test classes, not the full suite:

```bash
<test-filter-template with {NAME} = the wave's test classes>
```

Fix or report failures before committing. Catching a broken phase here is worth the one build: at Step 4 it would be buried under every later wave's code.

Then make a single commit for the whole wave:

```bash
git add -A && git commit -m "Wave: Phase <N> — <title>, Phase <M> — <title>"
```

**If two phases turn out to have edited the same file** (you'll see it as one agent's changes overwriting or interleaving with another's, or as unexpected diffs), stop and report it to the user. Parallel phases are not supposed to overlap, so an overlap means the wave grouping or the plan's dependency declarations were wrong — don't paper over it.

After committing, update the status table in the plan file to mark those phases ✅ Done.

---

## Phase Agent Prompt

Use this template when spawning an agent to implement a phase. Fill in every `<placeholder>`.
Paste the `invariants` from Step 0 verbatim into the "Cross-cutting invariants" block.

```
You are implementing Phase <N> of the plan at <relative-path-to-plan-file>.

## Your task
Implement exactly Phase <N>: <phase title>. Do not implement any other phase.

## Phase specification
<paste the full Phase <N> section from the plan verbatim>

## Cross-cutting invariants — hold these at all times
<paste the invariants section from .claude/docs-skills.md verbatim. If none are configured,
 write: "Follow the conventions already established in the surrounding code.">

## Design and architecture docs to read first
If this phase has an `Implements (design):` line, read those design-doc sections first — they are the
authoritative intent. Implement the *actual* design rule (its formula, values, and edge cases), not
just the plan's one-line paraphrase; the gap between the two is exactly what `audit-plan` later reports
as a divergence. Then read the <arch-docs-dir> files relevant to this phase. The plan's "How to use
this plan" section names which ones apply.

## Tests — write these BEFORE any production code

Your first act on this phase is to write its tests in <tests-dir>. Read a couple of existing test
files first and match the repo's framework, structure, and naming.

Write **one to three tests. No more.** Choose them from this phase's deliverables and its
`Implements (design):` rule — the behaviours this phase exists to produce. A good test here names a
promise the plan made, in terms a reader of the plan would recognise. Skip wiring, DI, registration,
constructors, trivial accessors, and anything whose failure the compiler would catch anyway: those
cost maintenance forever and assert nothing. This is not a coverage exercise. The tests you write are
a statement of what this phase was *for*.

Some phases have nothing to assert — pure scaffolding, wiring, a rename, a phase whose plan entry
says `Implements (design): none — infrastructure`. Write no test for those and say so in your final
report. A test invented to satisfy a checklist asserts that the code is the code, and it will be
maintained forever by people who assume it once meant something.

Write them from the **phase spec and the design rule it cites** — not from the implementation you are
about to write. That ordering is the entire point. A test written afterwards, by reading finished
code, restates whatever that code happens to do and carries its bugs forward as if they were
requirements; it cannot detect that the phase built the wrong thing. A test written from the spec
fails when the implementation and the intent disagree, which is precisely the failure worth catching.

<wave-mode>

Then implement the phase and bring the tests green. If, once you understand the code better, a test
turns out to assert the wrong thing, change it deliberately and say so in your final report. Do not
weaken an assertion, mark a test skipped/ignored, or delete it to reach green — a green suite that
was edited into greenness is worse than a red one, because it reports success.

## Definition of done
- The phase's tests are written, and written first — or the phase has no observable behaviour to
  assert and your report says so
- All deliverables in the phase spec are implemented
- Everything listed under "Deletes:" is removed
- The code is consistent (no mismatched references, no dead imports)

## Important constraints
- Do NOT update the plan's status table — the orchestrating skill handles that.
- Do NOT write documentation — that happens after all phases finish.
- Do NOT run any git commands — do not stage, commit, branch, or stash. Leave your changes in the
  working tree; the orchestrating skill commits after the wave completes. (Other phase agents may
  be editing other files at the same time, so any git operation from you would corrupt the shared
  index.)
- Stay strictly within the files your phase owns — your test files included. Name them after the
  code under test so they land in their own file. Independent phases in a wave are expected to
  touch disjoint files; if you find yourself needing to edit a file another phase clearly owns
  (a shared test fixture counts), stop and report it rather than editing it.
```

### The `<wave-mode>` block

Fill this into the prompt above depending on whether the agent shares its wave. It decides whether
the agent may touch the build at all.

**Alone in the wave** — the agent gets the full red-then-green cycle:

```
You are the only agent running right now, so you may run your own tests — but only your own. Run
them with `<test-filter-template with {NAME} = your test class>`, never the full suite, and confirm
they **fail before you write production code**.

Take that red seriously: it is what proves the test can detect the behaviour's absence. A test that
passes before the feature exists is asserting nothing, and one that merely fails to compile has told
you nothing either — get it to where it fails on its *assertion*, writing the type or signature it
needs if that's what it takes. Then implement, and run the same filtered command until green.
```

**Sharing the wave with other agents** — the agent must not build:

```
Other phase agents are working in this same directory right now, so do NOT invoke the build or the
test command. Concurrent builds collide on shared build output and fail for reasons that have
nothing to do with your code.

Write your tests first anyway, then implement — spec-first authoring is where most of the value is,
and it costs nothing to run. The orchestrator runs this wave's tests once, after every agent has
finished.
```

Running in parallel does cost you the red observation for those phases: nobody watches those tests
fail before the code exists. That is a real trade, made knowingly in exchange for wave concurrency.
If a phase is subtle enough that watching it go red is the point, give it a wave of its own.

---

## Step 4 — Verify the tests

The tests already exist — each phase wrote its own, first, back in Step 3. This step is the **full-suite run** plus an audit that green actually means something. Do it after every wave has been committed and the status table is fully updated. If the repo has no test suite (`test-command` blank in Step 0), skip to Step 5 and note it.

### 4a — Run the full suite

**If `tests-are-slow` is false**, just run the full suite (`test-command`), fix anything red, re-run. Done — go to 4b.

**If `tests-are-slow` is true**, the full suite is expensive, so minimize full runs. The most common way this skill wastes time is re-running the whole suite after every single edit while chasing one failure. Don't. Aim to run the *full* suite only twice: once to discover what's broken, once at the end to confirm no regression.

1. **Discover** — run the full suite once:

   ```bash
   <test-command>
   ```

2. If everything passes, go to 4b.

3. **Narrow** — if some tests fail, note their names, then iterate against *only those tests* using `test-filter-template` (substitute the failing test/class name for `{NAME}`). This skips the already-green tests and returns in seconds. Example shape:

   ```bash
   <test-filter-template with {NAME} = FailingClassOrMethod>
   ```

   Fix the code (or the test, if the test itself is wrong), re-run the filtered subset, repeat until that subset is green. This is where the loop lives — keep it narrow. (If `test-filter-template` is blank, just re-run the full suite, but as seldom as you can.)

4. **Confirm** — once the previously-failing tests pass, run the full suite **one** final time to catch any regression your fixes introduced elsewhere.

### 4b — Audit the phase tests

A green suite is only worth what its assertions are worth, and every phase agent had both the motive
and the means to make red go away by editing the test. Check that it didn't. Read what actually
happened to `tests-dir` across the wave commits:

```bash
git diff <commit-before-the-first-wave>..HEAD -- <tests-dir>
```

Three things to confirm:

- **Every phase left a test, or a reason it didn't.** Walk the phases, not the files: each one should
  have at least one test naming a behaviour from its spec, or an agent report explaining that the
  phase was pure scaffolding. A phase that produced observable behaviour and neither is a gap — write
  the missing test now, from the phase spec, exactly as Step 3 would have.
- **Nothing was disabled to reach green.** Look for tests deleted, renamed into irrelevance, or marked
  skipped/ignored (`[Skip]`, `@Ignore`, `.skip`, `xit`, commented-out bodies).
- **No assertion was hollowed out.** An assertion loosened until it cannot fail — a specific expected
  value swapped for "not null", a tolerance widened past the point of meaning, an exact string turned
  into a substring match — is the failure mode to watch for, because it leaves the test *looking*
  present in the diff.

Anything you find here, report it to the user with the phase it came from. Do not quietly repair it:
a test that had to be weakened is usually telling you the implementation and the plan disagree, and
that disagreement is the finding — it is exactly what `audit-plan` would otherwise catch much later.

### 4c — Stop if still broken

If failures remain after a reasonable fixing effort, stop and report every failure to the user rather than looping indefinitely. Do not continue to documentation or PR creation with a broken test suite.

---

## Step 5 — Update architecture documentation

After all phases are committed and the status table is fully updated, check the plan's "After implementation" section for which docs need updating (fall back to the docs under `arch-docs-dir` touched by the plan's changes).

- **If `has-update-arch-docs-skill` is true**: invoke the `update-arch-docs` skill for each affected subsystem — read its SKILL.md and follow its steps, or spawn an agent (with `model: "<worker-model>"`) to do so.
- **If false**: update the affected docs in `arch-docs-dir` directly — bring each one in line with what the code now does, matching the doc's existing structure. Don't invent new docs unless the plan calls for one.

---

## Step 6 — Create a PR

Skip this step if `pr-tool` is `none`; instead just push the branch and tell the user to open the PR themselves. Otherwise:

1. Ensure all changes are committed on the feature branch.
2. Push: `git push -u origin <branch-name>`
3. Write the PR body to a temp file (this keeps it identical across bash and PowerShell — no heredoc/here-string juggling), then create the PR with `--body-file`:

   Body content:
   ```
   ## Summary
   Implements <plan title> from <plans-dir>/<filename>.md.

   ## Phases implemented
   <bullet list: Phase N — description, one line each>

   ## Documentation updated
   <bullet list of updated docs, or "No documentation changes required">

   🤖 Generated with Claude Code
   ```

   - `gh`:  `gh pr create --title "Implement <plan title>" --body-file <tmpfile>`
   - `glab`: `glab mr create --title "Implement <plan title>" --description "$(cat <tmpfile>)"`

4. Check whether the PR has merge conflicts. If any, try to resolve them automatically.
5. Report the PR/MR URL and state whether there were conflicts and how you resolved them.

---

## Step 7 — Record the implementing branch in the plan

Stamp the plan file with the branch the work landed on, so anyone who later audits the plan (the `audit-plan` skill offers to clean up the implementing branch and worktree) can find and remove the right branch without guessing.

Read the current branch name:

```bash
git rev-parse --abbrev-ref HEAD
```

Append a short **Implementation record** section to the very end of the plan file:

```markdown
---

## Implementation record

- **Branch:** `<branch-name>`
- **PR:** <PR URL, or "none">
- **Implemented:** <YYYY-MM-DD>
```

If an "Implementation record" section already exists (the plan was executed in more than one pass), add a new bullet line under it rather than duplicating the heading.

Commit and push so the record travels with the PR:

```bash
git add -A && git commit -m "Record implementing branch in plan"
git push
```

(If `pr-tool` is `none`, still commit the record locally; skip the push if there's no remote.)

---

## Error and edge case handling

- **Phase fails:** stop the current wave and report the failure before proceeding to later waves. If it was a multi-phase wave, the other agents' edits are still uncommitted in the working tree — describe what landed so the user can decide whether to commit the partial wave, discard it (`git checkout -- .` / `git clean`), or retry the failed phase.
- **Two phases in a wave edited the same file:** stop and report it. The wave grouping or the plan's `Depends on:` declarations were wrong; don't auto-resolve.
- **A phase's test passes before the phase is implemented:** the red never happened, and that is information, not a formality to skip past. Either the test asserts something too weak to detect the behaviour's absence — rewrite it until it fails — or the behaviour already exists, in which case the phase is partly built already and the plan's "Current state" was wrong. Report the second case to the user rather than implementing over it.
- **All phases already Done:** report that nothing is left to implement and offer to just check docs and create a PR.
- **Only one phase in the whole plan:** skip the wave analysis. Just run it directly.
- **Plan not found:** list `plans-dir` contents and ask the user to pick one.
- **No config file:** proceed on the Step 0 defaults, but tell the user — a wrong `main-branch` or missing `test-command` is worth correcting before you start.
