---
name: audit-plan
description: >
  Audit a plan document from the project's plans directory on two axes: whether it was fully
  implemented in the codebase, and — when the project keeps a design corpus — whether that
  implementation actually accords with the design documentation it was supposed to realize. Use this
  skill whenever the user says "audit [plan name]", "check if X plan is done", "has [plan] been
  implemented?", "does the code match the design?", "audit this plan", "is the [plan name] plan
  complete?", "verify plan implementation", or points to a plan file and asks whether it's done or
  correct. After auditing, if the plan is fully implemented AND conformant with the design, prepend
  an audit note and move the file to the configured audited-plans archive, then offer to clean up the
  implementing branch and worktree (locally and remotely). Always invoke this skill when a plan file
  or plan name is mentioned alongside any question about its implementation status or design fidelity.
---

# Audit Plan Skill

Audit a plan document from the configured `plans-dir` on **two axes**:

1. **Implementation** — were the plan's deliverables actually built in the codebase? (always checked)
2. **Design conformance** — does the code actually realize the *design* the plan was implementing,
   or did it drift? (checked only when `design-docs-dir` is set)

A plan can be 100% implemented and still be *wrong*: every deliverable present, yet a formula ported
incorrectly, a placeholder left where the design demanded a real value, or a design rule that no
phase ever covered. The first axis catches "the plan wasn't finished"; the second catches "the plan
was finished but doesn't match what we designed." Both matter, and "done" now means both.

If — and only if — the plan is fully implemented **and** conformant, archive it to the configured
`audited-plans-dir` with a timestamped result note at the top.

---

## Step 0 — Read project config

Read `.claude/docs-skills.md` in the project root and extract the keys this skill uses; if the file
is missing, fall back to the defaults and tell the user no config was found.

| Key | Used for | Default if absent |
|-----|----------|-------------------|
| `plans-dir` | where plans live | `docs/plans/` |
| `audited-plans-dir` | archive destination (Step 5) | `<plans-dir>/../old_plans/audited/` |
| `design-docs-dir` | the design corpus to check conformance against (Step 3.5) | off — skip the design-conformance axis entirely |
| `main-branch` | the branch a plan may have been done directly on | `main` |
| `uses-worktrees` | whether to look for a `.claude/worktrees/<name>` dir in cleanup | `true` |
| `shell` | `bash` or `pwsh` for filesystem commands | detect from OS |

When `design-docs-dir` is `none` or absent, this skill performs the implementation audit only —
Step 3.5 is skipped and the verdict has a single axis, exactly as before the design chain existed.

Commands below are shown in bash. If `shell` is `pwsh`, translate the filesystem ones (e.g.
`mkdir -p` → `New-Item -ItemType Directory -Force`, `rm -rf` → `Remove-Item -Recurse -Force`,
`test -d` → `Test-Path`). Git commands are identical in both shells.

---

## Step 1 — Identify the plan file

The user will either:
- Name a plan (e.g., "audit district-resource-modifiers") — resolve to `plans-dir/<name>.md`
- Point directly at a file
- Paste the plan content inline

If ambiguous or multiple plans match, ask to clarify before proceeding.

Read `plans-dir/<plan-name>.md` in full before doing anything else.

---

## Step 2 — Understand what was planned

From the plan document, extract:

1. **Phases** — each phase is a distinct implementation block (e.g., "Phase 1 — Data model and config").
2. **Deliverables per phase** — the concrete code changes the phase requires:
   - New files to create (classes, structs, interfaces, modules)
   - Fields or methods to add to existing types
   - Fields or methods to remove (the plan explicitly says "delete" or "remove")
   - Files to delete entirely
   - Config/data files to update (fields to add or remove)
3. **Status table** — if present, note the current recorded status per phase (🔲/✅). Treat it as a hint, not ground truth — it may not have been kept up to date.
4. **Design anchors** (only if `design-docs-dir` is set) — the map you'll need for the conformance
   pass in Step 3.5. Extract:
   - the **Design source** header (which `systems/*.md` docs, and which sections, the plan claims to
     implement)
   - each phase's `Implements (design):` line
   - the **Design traceability** table (design rule → phase)
   - the `[design]`-tagged entries in **Locked decisions** (values inherited from the corpus)

   If `design-docs-dir` is set but the plan carries *none* of these (an older plan written before the
   design chain, or one that skipped Step 1.5), note it: you can't check plan-vs-design traceability
   that the plan never recorded, so Step 3.5 falls back to auditing the code directly against the
   design docs for the area the plan touched, and you should flag the missing traceability in the report.

Do not skip phases marked "Not started" in the status table. Verify each one independently.

---

## Step 3 — Verify each deliverable in the codebase

For every deliverable identified in Step 2, run appropriate searches. Do not guess — look.

**Files to create:**
- Use Glob to check whether the file path exists.
- If it should contain a specific type, also Grep for the type name.

**Fields / methods to add:**
- Grep for the identifier (field name, method name, property name) in the relevant file or across the codebase.
- A match confirms presence; no match means it was not added.

**Fields / methods to remove (the plan says "delete" or "remove"):**
- Grep for the identifier. Finding zero matches confirms removal; finding matches means it was not removed.
- Be careful with common words — scope the search to the relevant files.

**Files to delete:**
- Use Glob or Read to confirm the file no longer exists.

**Config / data file changes (JSON, YAML, TOML, etc.):**
- Read the relevant file and check both that the keys exist **and that their values match the plan spec**. Key presence alone is not enough — a wrong value (e.g. a placeholder where the plan requires a concrete id) can silently break runtime behaviour without any compile error.
- Pay particular attention to the plan's **Locked Decisions** section — those invariants must hold in the final implementation, not just structurally but in value.

Parallelise independent lookups (multiple Glob/Grep calls in a single message) to stay fast.

---

## Step 3.5 — Audit design conformance (only if `design-docs-dir` is set)

Step 3 answered "was it built?" This step answers a harder, more valuable question: **does what was
built actually do what the design says?** A deliverable can be present and still wrong — the method
exists but computes the wrong thing. Skip this entire step if `design-docs-dir` is `none` or absent.

**Read the design, then read the code that claims to implement it.** Open the `systems/*.md` docs
named in the plan's **Design source**. For each concrete item in their *Rules & formulas*, *Core
entities & data*, and *Primary interaction* sections, use the plan's **Design traceability** table
and per-phase `Implements (design):` lines to jump to the code that should realize it, then read that
code's actual logic. (If the plan recorded no traceability, search for the relevant code yourself and
note the missing anchors per Step 2.)

**Judge behavior, not identifiers.** The design doc is conceptual, not a schema — conformance is a
semantic judgment about what the code *does*, not whether a name matches. A `CalculatePrice` method
that exists but averages the wrong inputs is a divergence, not a pass. Read the formula in the design,
read the expression in the code, and decide whether they compute the same thing. Pay special attention
to the plan's `[design]`-tagged **Locked decisions**: those are concrete values the design pinned
(rates, thresholds, caps), and a wrong or placeholder value there is a silent runtime bug no compiler
will catch.

**Classify each design rule** into one of three outcomes:

- ✅ **Conformant** — the code realizes the design rule: same formula, same values, same behavior and
  edge cases.
- ⚠️ **Divergent** — the code for this rule exists (it was built) but does **not** match the design.
  Name the exact discrepancy and cite both sides. Common shapes:
  - *Wrong formula/logic* — design says clearing price is the supply/demand curve midpoint; the code
    uses the last trade price.
  - *Placeholder for a pinned value* — design pins `storage decay = 5%/turn`; the code has
    `decayRate = 0.0 // TODO`.
  - *Missing specified edge case* — design says a defecting manager keeps their district's garrison;
    the defection path transfers the garrison away.
- ❌ **Uncovered** — a design rule within the plan's claimed scope that **no** code realizes and no
  phase implemented. This is the design→plan→code gap the plan should have caught. (Only rules inside
  the **Design source** scope count here — you are auditing this plan, not the entire corpus. A system
  the plan never claimed to implement is out of scope, not "uncovered.")

**Direction matters.** Walk the *design doc's* rules and look for each in the code — that direction is
what surfaces ❌ Uncovered rules. Walking only the code would never reveal a rule that's simply absent.

Keep it evidence-based: for every ⚠️ and ❌, cite the design location (`doc §section`) and the code
location (`file:line`, or "absent"), and state the discrepancy in one sentence. Parallelise the
independent lookups.

---

## Step 4 — Produce the audit report

Report both axes, then a combined verdict.

**Axis 1 — Implementation (per phase).** For each phase use one of:

- ✅ **Complete** — all deliverables verified in the codebase
- ⚠️ **Partial** — some deliverables done, some missing (list what is missing)
- ❌ **Not implemented** — nothing from this phase is present

State the implementation verdict: **Fully implemented** (every phase ✅) / **Partially implemented**
(any ⚠️/❌) / **Not implemented** (all ❌).

**Axis 2 — Design conformance** (include this section only if `design-docs-dir` is set). Summarise
the Step 3.5 findings: list every ⚠️ **Divergent** and ❌ **Uncovered** rule with its design citation,
code location, and the one-sentence discrepancy. State the conformance verdict: **Conformant** (all
traced rules ✅) / **Divergent** (one or more ⚠️/❌). If the plan lacked traceability anchors, say so
here — the conformance check was best-effort against the code directly.

**Combined verdict.** "Done" requires both axes to pass:

- **Done** — fully implemented, and (no design corpus, or fully conformant). Eligible for archiving.
- **Not done — incomplete** — implementation gaps remain (list them).
- **Not done — diverges from design** — every deliverable is built, but the code doesn't match the
  design (list every divergence and uncovered rule). This is the case the second axis exists to catch;
  call it out plainly rather than letting a "fully implemented" headline imply the work is correct.

List all outstanding gaps — missing deliverables, divergences, and uncovered rules — so the user knows
exactly what remains before this plan is truly done.

---

## Step 5 — Archive if done

Only do this step if the **combined verdict** is **Done** — fully implemented, and either there is no
design corpus or the code is fully conformant with it.

**If the verdict is "Not done — diverges from design"** (every deliverable built, but one or more
⚠️/❌ conformance findings), do **not** archive automatically. A plan that ships code contradicting the
design is not finished, even though the status table says every phase is ✅. Present the divergences
and let the user choose:

- **Leave it open** (default) — the code or the design needs to change first. Stop here; nothing is
  archived. If the *design* was the thing that turned out wrong, point them at `design-doc` to update
  the corpus (and log the reversal) before re-auditing.
- **Archive anyway** — only on the user's explicit say-so, e.g. they've judged the divergence
  acceptable and will reconcile the design separately. If they choose this, record the accepted
  divergences in the audit note (5b) so the trail isn't lost.

Do not make this call yourself — divergence from an intended design is exactly the kind of thing a
human should decide on.

**5a — Ensure the target directory exists**

Check whether `audited-plans-dir` exists. If not, create it (`mkdir -p <audited-plans-dir>`).

**5b — Prepend the audit note**

First, find out which branch implemented this plan so the archive is self-documenting and Step 6
knows what to clean up. Check the plan's **Implementation record** section (the `execute-plan` skill
stamps the branch there). If it's absent, branch/worktree names are often random codenames, not the
plan name, so you cannot derive it — ask the user: "Which branch implemented this plan? (or say
'none' if it was done directly on `<main-branch>` or the branch is already gone)". Record their
answer for both the note below and Step 6.

Read the current plan file content. Prepend the following block at the very top (before the `#` title).
Include the **Design conformance** line only if `design-docs-dir` is set:

```
> **Audited:** <ISO date, e.g. 2026-06-27> — Fully implemented. All <N> phases verified in codebase.
> **Design conformance:** Conformant with <design docs> — all <M> traced rules verified.
> **Implemented by:** <branch name, or "unknown">

---

```

If the user chose "archive anyway" despite divergences, say so instead of claiming conformance, e.g.
`> **Design conformance:** Archived with accepted divergences — <one-line list>. See audit report.`

Keep everything else in the file unchanged.

**5c — Write and move**

Copy the old file to `audited-plans-dir/<filename>.md`, making sure the prepend from 5b is visible at
the beginning. DO NOT rewrite the whole file — only add the prepend block at the top.

Then delete the original file from `plans-dir`.

**5d — Commit the archive**

Stage both the deletion and the new archived file, then commit:

```bash
git add "<audited-plans-dir>/<filename>.md"
git add "<plans-dir>/<filename>.md"
git commit -m "audit: archive <filename> — fully implemented"
```

Confirm to the user: "Plan archived to `<audited-plans-dir>/<filename>.md` and committed."

---

## Step 6 — Clean up the implementing branch and worktree

Run this only after a successful archive (Step 5), and only if 5b recorded a real branch. A
fully-implemented, archived plan leaves its feature branch — and, if `uses-worktrees` is true and the
work happened in an agent worktree, its `.claude/worktrees/<name>` directory — as dead clutter. This
step removes them.

Deleting branches is destructive, and deleting a *remote* branch changes what everyone else on the
repo can see, so the step is deliberately cautious: it establishes the facts before it deletes, it
never force-deletes silently, and it asks before touching the remote. That care is the whole point —
a cleanup step that occasionally nukes live work would be worse than no cleanup step at all.

If 5b recorded "none" or "unknown", skip this step and tell the user there was nothing to clean up.

**6a — Establish the facts (one batch of read-only commands)**

- Current branch: `git rev-parse --abbrev-ref HEAD`
- Local branch exists? `git branch --list <branch>`
- Merged into the current branch? `git branch --merged` (is `<branch>` in the list?)
- Remote branch exists? `git ls-remote --heads origin <branch>`
- Worktree dir (only if `uses-worktrees`): the directory basename matches the branch's last path
  segment, e.g. branch `claude/loving-lederberg-451f57` → `.claude/worktrees/loving-lederberg-451f57`.
  Check if it exists.

Two hard stops — if either is true, report it and skip the affected deletion, since the archive
itself is already safe:
- The target branch **is the current branch** — you'd be sawing off the branch you're standing on.
- The worktree directory **contains the current working directory** — you'd be deleting the ground
  under your own feet.

**6b — Show the plan and confirm the remote**

Summarise exactly what exists and what you intend to do, for example:

```
Cleanup for plan <name> (branch <branch>):
  local branch  : present, merged ✓          → delete
  worktree dir  : .claude/worktrees/<name>    → delete
  remote branch : origin/<branch> present     → delete? (asking first)
```

Deleting the local branch and the worktree directory is local housekeeping — do those without a
separate prompt. Deleting the **remote** branch is visible to the whole team, so ask the user to
confirm that one action explicitly. If the remote branch doesn't exist (many hosts auto-delete the
head branch when a PR merges), say so and skip it — no confirmation needed for a no-op.

**6c — Delete**

Local branch — use the safe delete, which refuses if the branch isn't merged:
```bash
git branch -d <branch>
```
If `-d` refuses because the branch was **squash-merged** — its commits aren't ancestors of
`<main-branch>` even though its content landed there — do not silently force it. The audit already
proved the content is present, so say that, then offer `git branch -D <branch>` and let the user decide.

Worktree directory (only if `uses-worktrees`) — these are throwaway agent worktrees. Try the
git-native removal first in case it's a registered worktree, then fall back to deleting the plain
directory:
```bash
git worktree remove ".claude/worktrees/<name>" 2>/dev/null || true
test -d ".claude/worktrees/<name>" && rm -rf ".claude/worktrees/<name>"
```
(pwsh: `git worktree remove ".claude/worktrees/<name>"; if (Test-Path ".claude/worktrees/<name>") { Remove-Item -Recurse -Force ".claude/worktrees/<name>" }`)

Remote branch — only after the confirmation from 6b:
```bash
git push origin --delete <branch>
```

**6d — Report**

State precisely what was removed and what was left untouched, e.g. "Deleted local branch `<branch>`
and worktree `.claude/worktrees/<name>`. Remote branch was already gone. Nothing else touched."

---

## Tone and output style

- Be specific about what you checked and what you found — name the files and identifiers you searched for.
- If a deliverable is ambiguous (e.g. the plan says "add the method somewhere"), state your interpretation.
- Keep the audit report concise: one bullet per deliverable and one per divergence, not paragraphs.
- For design conformance, judge behavior, not names — read the code's logic against the design's rule.
- Do not archive unless the combined verdict is unambiguously **Done** (fully implemented *and*
  conformant, or conformant-not-applicable when there is no design corpus).
