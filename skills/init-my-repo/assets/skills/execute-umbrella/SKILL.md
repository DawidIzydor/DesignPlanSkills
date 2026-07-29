---
name: execute-umbrella
description: >
  Executes a multi-milestone **umbrella plan** — the plan file that declares no phases of its own and
  instead links out to milestone files (M1, M2, …). Reads the milestone table, groups milestones into
  waves by what each needs **merged** before it starts, creates one git worktree and branch per
  milestone, and dispatches one high-capability orchestrator agent per milestone; each of those runs
  the `execute-plan` skill inside its own worktree and fans out to cheaper per-phase agents. Merges
  each milestone's PR as its wave goes green and runs the integration suite on the merged result.
  Use this whenever a plan turns out to be an umbrella, or whenever the user says "run the whole
  batch", "execute all the milestones", "run the M1–M6 family", "do the umbrella plan", "implement
  the whole <name> family", or points `execute-plan` at a file carrying an "umbrella, not an
  executable plan" banner. If a plan file has a Milestones table and no phases of its own, this is
  the skill that runs it — `execute-plan` alone would find nothing to implement.
---

## What this skill does

1. Confirms the file really is an umbrella, and hands back to `execute-plan` if it isn't
2. Reads the **Milestones** table and groups milestones into **waves** by their `Runs after:` gate
3. Presents the wave plan, the agent count, and the merge policy — and takes **one** confirmation for the whole run
4. Per wave: creates a worktree and branch **itself** for each milestone, then dispatches one milestone agent per milestone
5. Rides out the stall that milestone agents reliably hit when they fan out
6. Verifies, merges, and integration-tests each wave before the next wave's worktrees are cut
7. Cleans up, stamps the umbrella, and reports

## The shape of the run

```
this session  (orchestrator — milestone waves, worktrees, merges)
   ├── milestone agent M1   <milestone-model>   in .claude/worktrees/<slug>-m1
   │      ├── phase agent   <worker-model>
   │      └── phase agent   <worker-model>
   └── milestone agent M3   <milestone-model>   in .claude/worktrees/<slug>-m3
          └── phase agent   <worker-model>
```

Three levels, and the model tier drops at each one for a reason. Wave scheduling, merge ordering and
recovery are the judgment-heavy parts and stay in this session. A milestone is a whole `execute-plan`
run — dependency analysis, a test audit, a docs pass, a PR — which is why it gets a capable model.
A phase is mechanical fan-out against a written spec, which a cheaper model does well and fast.

The middle tier is what makes an umbrella worth running this way at all: a milestone's phases are
**serial or near-serial**, so running six milestones in one session means one long chain, while
running them as separate agents lets the independent lines of the graph proceed at the same time on
separate branches — which is exactly what the umbrella's own dependency diagram is telling you to do.

---

## Step 0 — Read project config

Read `.claude/docs-skills.md` in the project root. This skill uses everything `execute-plan` uses
(it passes the config down) plus two keys of its own.

| Key | Used for | Default if absent |
|-----|----------|-------------------|
| `plans-dir` | where plans and milestone files live | `docs/plans/` |
| `main-branch` | merge target; the branch never committed to directly | `main` |
| `branch-prefix` | prefix for each milestone's branch | `plan/` |
| `pr-tool` | `gh` / `glab` / `none` — how PRs open and merge | `gh` if on PATH, else `none` |
| `milestone-model` | model for the per-milestone orchestrator agents | `opus` |
| `worker-model` | model for the phase agents those orchestrators spawn | `sonnet` |
| `worktrees-dir` | where this skill creates milestone worktrees | `.claude/worktrees/` |
| `test-command` | the integration run after each wave merges | ask the user |
| `shell` | `bash` or `pwsh` for filesystem commands | detect from OS |

Commands below are shown in bash; translate filesystem commands if `shell` is `pwsh`. Git, `gh` and
`glab` are identical in both.

**Check `worktrees-dir` is ignored** (`git check-ignore <worktrees-dir>`). If it isn't, add it to
`.gitignore` before creating anything — an un-ignored worktree directory turns every later
`git add -A` into a catastrophe.

---

## Step 1 — Confirm this really is an umbrella

Getting this wrong in either direction wastes a lot of work, so decide it on structure rather than on
the filename or a hunch.

Read the plan file. It is an umbrella when **both** hold:

- It **declares no phases of its own** — no `## Phase N` sections, no per-phase status table
- It carries a **Milestones table** linking out to sibling plan files, usually with a `Runs after`
  column, and usually under a banner saying it is not executable

The phase test is the decisive one. The banner is a courtesy that an older or hand-written umbrella
may lack; a file with real phases is an ordinary plan whatever it calls itself.

- **Not an umbrella** → say so and run the `execute-plan` skill on it instead. Read
  `.claude/skills/execute-plan/SKILL.md` and follow it; don't reimplement it here.
- **A single milestone file** (it has phases *and* a `Runs after:` header) → also `execute-plan`.
  That is the normal way one milestone gets run, and this skill is only for running several.
- **Ambiguous** → show the user what you found and ask, rather than guessing. Dispatching a fleet at
  the wrong file is expensive; asking costs one message.

---

## Step 2 — Build the milestone waves

Read the Milestones table and the prose around it. For each milestone extract: number, file path,
what it covers, its `Runs after` gate, and its status.

Skip milestones already marked done — but **verify** rather than trusting the table: confirm the
work is actually on `main-branch` (`git log --oneline origin/<main-branch> | grep`, or check the
linked PR merged). A milestone whose link points into the audited-plans archive is strong
corroboration; a ✅ with the plan still sitting in `plans-dir` deserves the git check. A status table
that drifted ahead of reality is the one input here that silently corrupts everything downstream,
because a later milestone gets cut from a base that lacks its predecessor.

### A milestone whose status is a finding, not a state

An audited family often carries rows like *"⚠️ Open — the derived export reserve is unwired"*. That
milestone **shipped**; what's open is a divergence an audit found afterwards. Its own phase table is
all ✅, so dispatching an agent at it does the worst possible thing — `execute-plan` skips every
finished phase, finds nothing to implement, and reports success while the actual gap goes untouched,
because the gap is described in the *umbrella's* audit note and nowhere in the milestone file.

Never dispatch one of these blind. List them for the user with their findings and ask how to
proceed: the honest answer is usually that each finding needs `plan-feature` to turn it into real
phases first. Milestones that are genuinely 🔲 not started run normally alongside that conversation.

### The gate is "merged", not "started"

`Runs after: M1 merged` means M2's worktree cannot even be **created** until M1 is on
`main-branch` — M2's phases were written against a codebase that includes M1's. This is the whole
reason this skill exists as a wave scheduler rather than a fan-out: the graph has real barriers in
it, and cutting a worktree early bases a milestone on a tree its plan does not describe.

Assign each milestone to the earliest wave where every gate it names sits in an earlier wave. Read
`Runs beside:` as corroboration, but let the `Runs after` graph be authoritative — if they disagree,
say so and ask.

### Harvest the collision notes

An umbrella that follows the house format names, in prose, the files where a collision between
concurrent milestones is expected, and which milestone owns each — *"the one file where a collision
between M4 and M5 is expected is `ShellLayout.cs`; M4 owns it, M5 should not touch it."*

Extract these now. They become two things later: a **do not touch** line in the non-owner's brief
(Step 5), and the **merge order** within the wave (Step 7). They are the single most useful sentence
in the umbrella, and they are easy to skim past.

---

## Step 3 — One confirmation that covers the run

Present the plan and get one approval for the whole thing. Ask once because a run like this may span
hours; stopping at every seam defeats the point. Ask at all because merging a PR is outward-facing
and irreversible, and because the agent count has real cost.

```
Umbrella:  <title>  (<N> milestones, <M> outstanding)

Wave 0 [concurrent]:  M1 <title>   M3 <title>
Wave 1 [concurrent]:  M2 <title>   M4 <title>   M5 <title>
Wave 2:               M6 <title>

Agents:   <milestone-model> × <count in the widest wave>, each fanning out to
          <worker-model> phase agents (roughly <n> per milestone)
Branches: <branch-prefix><slug>-m<N>, one per milestone, each PR into <main-branch>
Merges:   I merge each milestone's PR once its own suite is green, then run
          <test-command> on the merged <main-branch> before cutting the next wave.
```

Then ask plainly whether to proceed with auto-merge, or whether they'd rather review each PR. Their
answer governs the whole run, and you don't ask again — but do report each merge as it happens, so
"I approved this run" never means "I lost sight of it".

If they decline auto-merge, everything else in this skill still applies; you simply stop at the end
of each wave and wait for them to merge before Step 4 of the next one.

### The housekeeping branch

The umbrella's status table needs updating as waves complete, and it belongs on a branch like
everything else. Create one now, off `main-branch`, and reuse it for the whole run:

```bash
git checkout -b <branch-prefix><umbrella-slug>-status
```

Commit and push the status flip after each wave, and open its PR at the end (Step 8). Pushing each
wave rather than batching means that if the session dies mid-run, the record of what already merged
survives somewhere other than this conversation.

Return to `main-branch` before cutting worktrees.

---

## Step 4 — Prepare a wave: you create the worktrees

Bring `main-branch` up to date first — the previous wave's merges are the base this one builds on:

```bash
git checkout <main-branch> && git fetch origin && git pull
git rev-parse HEAD          # note this sha; it is the wave's base
```

Then create one worktree per milestone in the wave, **yourself**, from that explicit sha:

```bash
git worktree add <worktrees-dir>/<slug>-m<N> -b <branch-prefix><slug>-m<N> <base-sha>
```

**Do not use the `Agent` tool's `isolation: "worktree"` for this.** That option bases the worktree on
the commit *this session started on*, not on the branch you name in the prompt and not on current
HEAD — so on a chained family every agent after the first silently works against a tree missing its
predecessor's work, and you find out when every file conflicts. Creating the worktree here and
handing the agent an absolute path removes the failure mode rather than warning about it, and it
leaves you holding the branch names you need at merge time.

Verify each one before dispatching — thirty seconds now against an hour of misapplied work:

```bash
git -C <worktrees-dir>/<slug>-m<N> rev-parse --abbrev-ref HEAD    # expect the milestone branch
git -C <worktrees-dir>/<slug>-m<N> rev-parse HEAD                 # expect the base sha
```

---

## Step 5 — Dispatch the milestone agents

Spawn one agent per milestone in the wave, **all in a single message** so they run concurrently.

- `model: "<milestone-model>"`
- `subagent_type`: a **general-purpose** type — one whose tool set includes the `Agent` tool. This
  matters more than it looks: a milestone agent that cannot spawn will not report that it can't, it
  will quietly implement every phase itself, serially, on the expensive model. If you're unsure which
  types can spawn, check the agent-type list before dispatching rather than after.
- **No `isolation`** — the worktree already exists and the agent is being pointed at it by path.

### Milestone Agent Prompt

```
You are executing milestone M<N> of the umbrella plan <path-to-umbrella>.

## Your worktree — verify this before anything else
Your working directory is: <absolute-worktree-path>
It is already created and already on the right branch. Prefix every shell command with
`-C <absolute-worktree-path>` (git) or cd there first, and confirm both of these before you start:

    git -C <absolute-worktree-path> rev-parse --abbrev-ref HEAD   -> <branch-prefix><slug>-m<N>
    git -C <absolute-worktree-path> rev-parse HEAD                -> <base-sha>

If either disagrees, stop and report it rather than working — you would be building against the
wrong base, and every file you touch would conflict at merge.

Do NOT create a worktree or a branch of your own. Do NOT work in the main checkout.

## Your task
Run the `execute-plan` skill against exactly this milestone file: <path-to-milestone-file>
Read <absolute-worktree-path>/.claude/skills/execute-plan/SKILL.md and follow it end to end. It
owns the phase-level machinery — waves, the tests-first phase agents, the test audit, the docs pass,
the PR. Follow it as written, with these four deltas:

1. You are already on a dedicated branch in a dedicated worktree. Its Step 2 branch check should
   find nothing to do; do not create another branch.
2. Spawn your phase agents with `model: "<worker-model>"`, as that skill already instructs. Do not
   propagate your own model down to them — the tier split is deliberate.
3. Open your PR against <main-branch>. Never against a sibling milestone's branch, even if that
   milestone is your prerequisite: a PR stacked onto another PR's branch strands its commits when
   the parent squash-merges.
4. Do NOT merge your own PR, and do not update the umbrella's milestone table. The orchestrating
   session merges, in an order it controls, after checking the wave as a whole.

## Files you do not own
<for each collision note in the umbrella naming a file owned by a concurrent sibling:>
`<file>` belongs to M<owner> this wave. Do not edit it. If your work seems to need it, stop and
report that instead — the umbrella predicted this file as the collision point, so needing it means
either the prediction or your approach is wrong, and both are worth knowing before a merge conflict.

## Prerequisites already met
<name the milestones merged into your base, one line each, and what each delivered — so the agent
 recognises the code it finds rather than treating it as unexpected.>

## When you finish, report exactly
- Branch name, and confirmation everything is committed and pushed
- PR URL, and the base branch it targets
- Which phases landed, and any that didn't, with the reason
- Test status: what you ran and what it said
- Anything you deliberately left undone, and any place the milestone file disagreed with the code
```

---

## Step 6 — Expect the stall

**A milestone agent will very likely end its turn the moment it dispatches a parallel phase wave**,
saying something like "waiting for all four" or "I'll pick this up when they return". Nothing is
running for it and nothing will wake it: a subagent is only resumed if its child returns while it is
still inside its turn. On one eight-milestone family this hit two of the three orchestrators that
fanned out to a parallel wave — including ones warned about it in their own brief. Warning helps; it
does not prevent.

So treat the early return as a normal state of the protocol, not as failure:

- **The work is not lost.** It is sitting uncommitted in that agent's worktree, on the right branch.
  Confirm with `git -C <worktree> status --short` before doing anything else.
- **The wake signal comes to you.** The grandchildren's task notifications arrive at *this* session,
  not at the stalled agent. When the last phase agent for a milestone reports, resume its
  orchestrator with `SendMessage` to its agent id, carrying:
  - a summary of what each phase agent reported
  - **"do not spawn them again — their work is already in your worktree"**
  - **"re-derive your state from `git status` and `git diff`, not from memory"** — resumed agents'
    recollections have disagreed with their own worktrees about what had landed
- **Check liveness without polling.** `git -C <worktree> status --short` plus the mtime of the
  agent's transcript tells you the difference between working and stalled. Don't sleep-loop.
- **Never work in a worktree while its agent is also resuming.** Two writers in one worktree
  produced files mutating between another agent's Read and Edit, and a transient false red. Either
  resume the agent or do the slice yourself — not both.

If an agent is genuinely dead rather than stalled (transcript stale, worktree untouched, no PR), the
worktree is still a complete record of where it got to. Dispatch a fresh agent at the same path with
the same brief plus a "here is what is already done" preamble derived from `git log` and `git diff`.

---

## Step 7 — Close the wave: verify, merge, integrate

Only when every milestone in the wave has reported.

### 7a — Verify each milestone before merging anything

- Worktree clean and fully pushed: `git -C <wt> status --short` empty, and
  `git -C <wt> log origin/<branch>..HEAD` empty
- The PR exists and **targets `main-branch`** — check, don't assume; a PR retargeted at a sibling
  branch is the mistake that strands a whole milestone's commits on squash-merge
- Its own suite was green, per the agent's report
- Nothing was left undone that the next wave depends on

A milestone that fails verification does **not** merge. Report it, and hold back only the milestones
whose gate it is — the rest of the wave merges normally. Stopping the whole run for one bad
milestone wastes the concurrency you just paid for.

### 7b — Merge in the order the collision notes imply

Within a wave, **merge the owner of a shared file before the milestones that were told not to touch
it**. The owner's version is the one that should land first; the others were built to avoid it, so
their diffs stay small and any conflict is the trivial kind. Merging the other way round makes the
owner's substantial change the conflicting one.

Remove the worktree first (its work is merged into the PR and pushed; the directory is now just a
lock on the branch name), then merge:

```bash
git worktree remove <worktrees-dir>/<slug>-m<N>
gh pr merge <number> --squash --delete-branch
git checkout <main-branch> && git fetch origin && git pull
git merge-base --is-ancestor <milestone-branch-tip> HEAD && echo "landed on <main-branch>"
```

That last check is worth the line. It is what catches a PR that merged somewhere other than where
you thought, which is otherwise invisible until a later milestone is cut from a base missing it.

**If a PR has conflicts**, don't resolve them from the outside — `SendMessage` its milestone agent
(before removing the worktree). It holds the context for which side of each hunk is right. Verify
the resolution yourself before merging.

### 7c — Run the integration suite on the merged result

```bash
<test-command>
```

Each milestone tested its own branch in isolation. **The combination of two concurrent milestones has
never been tested by anyone until this moment** — this run is the only place cross-milestone breakage
can surface before it reaches the next wave, and finding it here means it's attributable to a pair,
not buried under three more milestones.

If it's red, fix it here on `main-branch`'s successor branch — do not cut the next wave's worktrees
from a broken base. Report the failure and what caused it; a break that only appears when two
milestones combine is usually telling you the umbrella's "no shared files" claim was wrong, which is
a finding about the plan and not just a bug.

### 7d — Stamp progress

On the housekeeping branch, flip the merged milestones to done in the umbrella's table, with their PR
links. Commit and push, then return to `main-branch`. Then go back to Step 4 for the next wave.

---

## Step 8 — Close the run

1. **Prune worktrees**: `git worktree prune`, then `git worktree list` to confirm nothing is left
   behind. `audit-plan` later offers to clean these up, and it can only do that if they're gone or
   findable.
2. **Stamp the umbrella** with an Implementation record — one line per milestone: branch, PR URL,
   date merged. This is the trail anyone auditing the family follows.
3. **Open the housekeeping PR** for the status-table and Implementation-record commits, and merge it.
4. **Report**, as a table:

```
| Milestone | Branch | PR | Phases | Status |
```

Plus, in prose: anything a milestone agent flagged as undone or as a disagreement between its plan
and the code, any integration failure and how it was fixed, and any milestone that didn't run and
why. Those are the things the user cannot reconstruct from the PR list, so they're the part of the
report that earns its place.

---

## Error and edge case handling

- **Not an umbrella** — hand off to `execute-plan` (Step 1). Don't reimplement phase execution here.
- **One milestone outstanding** — skip the wave machinery entirely. Run `execute-plan` on it directly
  in this session; a worktree and a dispatched agent buy nothing when there's no concurrency to win.
- **A milestone's status says done but nothing is on `main-branch`** — stop and ask. Either the table
  drifted or a PR was closed unmerged, and both change the wave graph.
- **Every outstanding milestone is a reopened one** (Step 2 — shipped, then an audit found a gap).
  There is nothing here to dispatch. Say so, summarise the findings, and offer `plan-feature` to
  turn them into phases. Running the fleet anyway produces a clean report about no work.
- **`Runs after` and `Runs beside` contradict each other** — surface both readings and ask. Guessing
  here either serialises work that could be concurrent or cuts a worktree from an incomplete base.
- **A milestone agent reports it needs a file another milestone owns** — that's the collision the
  umbrella predicted. Don't let it edit the file. Either wait for the owner to merge and rebase, or
  take it back to the user: two milestones needing one file means the cut was wrong.
- **`pr-tool` is `none`** — there is nothing to merge. Have milestone agents push their branches, and
  merge locally with `git merge --no-ff` into `main-branch` in the same order Step 7b describes,
  running the integration suite the same way. Tell the user, since the review step disappears.
- **The user declined auto-merge** — run everything else identically, stopping at the end of each
  wave to hand them the PR list and wait. Re-verify (7a) after they merge, since they may have merged
  a subset or in a different order.
- **A wave is wider than you want to run at once** — cap it, dispatch the rest as a follow-on wave,
  and **say which milestones you deferred and why**. A silently narrowed wave reads as a completed
  one in the final report.
