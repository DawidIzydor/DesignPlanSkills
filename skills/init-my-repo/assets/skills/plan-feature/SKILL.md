---
name: plan-feature
description: >
  Creates, updates, or restructures a feature plan for the current project. Use this skill
  whenever the user wants to plan a new feature, design a subsystem, decide how to migrate
  something, or think through a multi-phase implementation before starting to code. Also use it
  when an existing plan has grown too large to execute in one session and needs splitting into
  milestone-level plans — phrases like "split this plan", "break the plan into milestones", or
  "this plan is too big for one session".
---

## What this skill does

Writes a structured plan that a future agent can pick up cold and execute one phase at a time —
without needing to re-derive current state, re-litigate design decisions, or read the full
conversation history.

When the project keeps a design corpus (`design-docs-dir` in the config), the plan is the concrete
implementation of an **already-decided design**: it turns the design's *what and why* into a
buildable *how*, and stays traceable back to the design so the shipped code can later be audited
against it. When there is no design corpus, the plan is derived from the conversation and code as
usual.

A good plan is **opinionated**. It makes decisions so the implementing agent doesn't have to
improvise, documents why, and clearly labels what's still open. It is not a requirements
document or a wishlist. Be clear about *which* decisions a plan is entitled to make: the plan owns
**implementation** decisions (module boundaries, contracts, ordering, what to delete). When a design
corpus exists, **design** decisions (the rules, formulas, and behaviors themselves) are inherited
from it — cited, not reinvented. A design gap you discover while planning goes back to `design-doc`,
not resolved silently in the plan.

The plan contains **only the work that will actually be built in this pass**. Ideas that come up
but are deliberately not being built now — future extensions, nice-to-haves, things blocked on
other work, explicitly descoped tangents — do not belong in the plan. They go to the backlog
(Step 3.5). Keeping them out is what lets the implementing agent trust that every phase in the
plan is a phase to execute, not a phase to second-guess.

---

## Step 0 — Read project config

Before anything else, check for `.claude/docs-skills.md` in the project root.

- If the file **exists**: read it and extract:
  - `plans-dir` — where plans are written (required)
  - `backlog-dir` — where deferred/out-of-scope items are parked (used in Step 3.5). If the key is
    absent, fall back to `<plans-dir>/../backlog/`
  - `arch-docs-dir` — where architecture docs live, used in Step 2 for ground truth reading
  - `design-docs-dir` — the upstream design corpus this plan implements. If set (not `none`), it
    turns on Step 1.5 and the design-traceability output. If the key is `none` or absent, the
    design steps are skipped entirely.
  - `invariants` — the "Cross-cutting invariants" section: rules the implementing agent must
    maintain throughout any plan (e.g. threading constraints, config conventions, naming rules)
- If the file **does not exist**: use these defaults and note to the user that no config was found:
  - `plans-dir` → `docs/plans/`
  - `backlog-dir` → `docs/backlog/`
  - `arch-docs-dir` → `docs/architecture/`
  - `design-docs-dir` → off (no design corpus; skip Step 1.5 and the traceability section)
  - No invariants (omit the "cross-cutting invariants" section from the plan)

All subsequent steps use the paths and invariants resolved here.

---

## Step 1 — Understand what's being planned

Read the conversation. Identify:

- **The goal**: what capability or structural change is being introduced?
- **The scope**: is this a new subsystem, a migration, an extension of something existing?
- **Constraints mentioned**: things the user said must or must not happen
- **Decisions already made**: design choices the user committed to during discussion
- **Open questions**: things not yet decided

If the scope is unclear, ask one focused question before proceeding.

Check `plans-dir` for any existing plan that covers this area. If one exists, update it (add
phases, revise status, record new decisions) rather than creating a parallel file.

---

## Step 1.5 — Anchor to the design (only if `design-docs-dir` is set)

If `design-docs-dir` is set, the plan is not derived from the conversation alone — it is the
concrete implementation of an already-decided design. Make that design the backbone before you
write a single phase:

- **Find the system doc(s).** Locate the `systems/<system>.md` under `design-docs-dir` that covers
  what's being planned, plus any neighbours named in its *Interactions* section. Read them in full.
- **Check readiness.** A design doc that isn't `design-locked` is still moving. If the target system
  is `exploring` or `drafting`, stop and tell the user what is still open — planning against a
  drafting design bakes in decisions the design owner hasn't actually made. Proceed only if they
  explicitly ask for a provisional plan, and label it as such at the top.
- **Extract the intent to implement.** From the doc's *Rules & formulas*, *Core entities & data*,
  and *Primary interaction*, pull the concrete things the code must do — the formulas with their
  real values, the entities and their attributes, the behaviors and their edge cases. These are the
  raw material for your phase deliverables, and each is a line `audit-plan` will later check the
  code against.
- **Plan the HOW, never re-decide the WHAT.** Turn the design's *what/why* into a concrete *how* —
  module boundaries, contracts, ordering, what to delete. Do **not** make design decisions here. If
  you hit a genuine design gap or contradiction the corpus hasn't resolved (a rule with a `[placeholder]`
  value, two docs that conflict, a behavior no doc specifies), do not invent an answer in the plan.
  Name it to the user and route it back to `design-doc`. A plan that silently invents design is
  exactly what `audit-plan` will flag as divergence later — so surface it now, while it's cheap.

If `design-docs-dir` is `none` or absent, skip this step.

---

## Step 2 — Ground truth: read current state

Before writing the plan, understand what exists *now*. This section of the plan ("Current state")
is its most perishable asset — it must reflect actual code, not memory or assumptions.

If you anchored to a design in Step 1.5, the **gap** between that intended behavior and the current
code *is* the work this plan schedules. Read current state with the design in hand: what does the
design require that the code doesn't do yet, and what does the code do that the design has moved
away from (and must therefore be changed or deleted)?

- Read the relevant architecture docs in `arch-docs-dir`
- Grep for the key types and services involved
- Note what already works, what's dead/legacy, and what's genuinely missing
- Note any naming or contract drift between what the architecture docs say and what the code says

If you find that something the user mentioned has already been implemented (or was implemented
differently than discussed), flag it — the plan must reflect reality.

**Before checking any code** read the architecture docs and ask questions to the user.
This will speed your reasoning as the user knows answers to most of your questions.

---

## Step 3 — Write the plan

Write to `plans-dir/<kebab-case-name>.md`. Match the style of the existing plan files.

> **If the phase list is running long — roughly 6+ phases — read Step 4 before you write the phase
> bodies.** A plan that wants splitting into milestones is far cheaper to write that way from the
> start than to write whole and cut up afterwards.

### Required sections (in order)

**Title and one-paragraph summary** — what this plan covers and what the end state looks like.
If a major area is explicitly excluded, say so up front.

**How to use this plan** (if phases exist) — tell the follow-up agent how to work from it:
one phase per session, decisions-at-time-of-implementation for anything marked open, etc.

If invariants were defined in `.claude/docs-skills.md`, include them here under a
**Cross-cutting invariants** subsection. If no invariants are configured, omit this subsection.

**Design source** (only if `design-docs-dir` is set) — name the design doc(s) this plan implements
and the state they were in when you planned, e.g. `Implements systems/market.md (design-locked,
last updated 2026-07-10) and systems/economy.md §Prices`. This is the contract `audit-plan` checks
the code against, so be precise about which docs and which sections. Omit this section entirely when
there is no design corpus.

**Current state (verified)** — a compact, factual snapshot of what exists right now. Verified
means you read the code. Use past tense or present tense, never future. Mark anything unverified
with `[unverified]`.

**Locked decisions** — choices that are settled and should not be re-opened. For each: state the
decision, the reason, and if there's a rejected alternative worth noting. Bullet list, not prose
paragraphs. When a design corpus exists, tag each decision's provenance so an auditor knows where to
check it: `[design]` for a decision inherited from the corpus (cite the doc — `audit-plan` verifies
these against the design, not just the code) versus `[impl]` for an implementation decision this plan
is making itself (an auditor checks these against the code only). Without a design corpus, all
decisions are `[impl]` and the tag can be dropped.

**Phases** — one `##` section per phase. Each phase should:
- Be independently deliverable (compiles and works on its own)
- State its dependency on other phases (`Depends on: Phase N`, or `Depends on: nothing`)
- List what gets deleted, not just what gets created — deletions are explicit deliverables
- Have an "Open details for the session" subsection for things intentionally left to the
  implementing agent (things where the session context will inform the right answer)
- When `design-docs-dir` is set, carry an `Implements (design):` line naming the specific design
  rules the phase realizes — e.g. `Implements (design): systems/market.md §Rules — clearing-price
  formula and the price-floor rule`. Keep it specific enough that an auditor can pull that exact
  rule from the doc and check the code computes it. A phase that implements no design rule (pure
  scaffolding, wiring) should say `Implements (design): none — infrastructure`.

> **Write the `Depends on:` line on every phase.** The `execute-plan` skill parses exactly this
> line to decide which phases can run in parallel. A missing or vague dependency declaration is
> what makes two agents collide on the same file. Be explicit: name the phase numbers, or say
> `nothing`.

**Dependency diagram** — an ASCII ordering diagram showing phase dependencies. Always include
this if there are 3+ phases.

**Status table** (optional but recommended) — a markdown table of all phases with emoji status
(✅ Done / ⏳ In progress / 🔲 Not started / ⛔ Blocked — reason).

**Design traceability** (only if `design-docs-dir` is set) — a compact table mapping each design
rule to the phase that implements it, so the design→plan mapping is auditable at a glance and gaps
are obvious:

```
| Design rule (doc §section)                     | Phase | Notes                          |
|------------------------------------------------|-------|--------------------------------|
| systems/market.md §Rules — clearing price      | 2     | formula ported verbatim        |
| systems/market.md §Rules — price floor         | 2     |                                |
| systems/economy.md §Entities — Ledger balance  | 1     |                                |
```

Build this by walking the design doc's *Rules & formulas* and *Core entities & data*, not by
walking your phases — that direction is what surfaces a design rule **no phase covers**. If you find
one, either add a phase for it or, if it's deliberately out of scope for this pass, note it as
deferred (Step 3.5) and say so here. An uncovered rule left unremarked is a plan-vs-design gap
`audit-plan` will report against you.

### What to leave open vs. lock down

Lock down: the overall structure, the public contracts (interface names and methods), the
inter-service communication seams, what gets deleted, the ordering constraints.

Leave open: internal implementation details, exact method bodies, edge-case handling that
depends on what the code looks like when you get there. Mark these explicitly with
**"Open detail for the session:"** so an agent knows it's allowed to decide, not looking for a
pre-existing answer.

### Voice and tone

- **Address the implementing agent directly**, not the user. "Read `<subsystem>.md` before
  starting" not "The developer should read…"
- **Imperative.** "Delete `LegacyResolver` once the new path covers it." Not "It may be
  possible to delete…"
- **No hedging.** If a decision is locked, state it flatly. If it's open, say so explicitly.
  Never hedge a locked decision.
- **Concise.** A plan is a briefing, not a design essay. Enough detail to execute, no more.

---

## Step 3.5 — Route deferred items to the backlog

As you write the plan, you'll accumulate ideas that are real but out of scope for this pass.
Instead of parking them in an "Out of scope" or "Future work" section *inside* the plan, move
them to the backlog so the plan stays a clean list of work to do.

**What goes to the backlog** — anything raised during discussion or discovered in ground-truth
reading that you are consciously choosing *not* to build in this plan:
- future extensions that build on seams this plan leaves behind
- nice-to-haves and polish that aren't required for the end state
- work blocked on something else landing first
- tangents the user explicitly descoped ("not now", "separate effort")
- a smaller defect or cleanup you noticed but that isn't this plan's job

**What stays in the plan** — do not confuse *deferred work* with an *open detail*. An "Open
detail for the session" is part of a phase that **is** being built; its implementation is just
left to the agent's judgment at build time. That stays in the phase. Only move things that will
**not** be built by this plan at all.

**How to write a backlog entry.** Read the backlog before appending so you match its structure
and don't duplicate an existing item.
- Small items: append to `backlog-dir/backlog.md` under the existing subsystem group they fit,
  or start a new group if none fits. Match the file's existing heading depth and entry style.
- A large, self-contained deferred epic: give it its own file in `backlog-dir/` (mirror how
  existing standalone backlog files are structured) and add a one-line pointer from
  `backlog.md`.
- Every entry records **where it came from** so the trail survives: a `Source:` line naming this
  plan (and the phase/seam it was deferred from, if relevant) and a `Docs:` line pointing at the
  relevant architecture docs — following whatever trailer convention the backlog already uses.

**Cross-link, don't re-describe.** The plan may carry at most a one-line pointer ("Deferred
extensions live in the backlog") if it helps orient the reader. It must not re-explain the
deferred items — that content now lives in the backlog, and duplicating it defeats the purpose.

---

## Step 4 — Split into milestone plans when the plan outgrows one session

`execute-plan` runs **a whole plan file in a single session** — every phase, wave by wave, then the
test pass, the docs update, and one PR. So the *plan file*, not the phase, is the unit that has to
fit in a session. When a plan outgrows that, nothing fails loudly; it degrades quietly. The last
phases get a session that already spent its context on the first ones, the status table drifts out
of sync with the work, and a single PR lands carrying a dozen phases nobody can review.

So when a plan is large enough that its phases form **logical milestones**, split it: each milestone
becomes its own plan file that runs standalone as one `execute-plan` session, and the original file
stays behind as an **umbrella** that owns the shared context.

### When to split

Both of these have to be true:

- **It's genuinely large** — roughly **6+ phases**, or a shared preamble (Current state, Locked
  decisions, Design source) heavy enough that a session reads for a long time before it can touch code.
- **The cuts are real** — the phase graph has seams where a group of phases finishes something whole.
  A milestone that can't compile, ship, and be reviewed on its own is not a milestone.

A four-phase plan stays one file even if it's wordy. A ten-phase plan where every phase depends on
the one before it is a chain, not milestones — leave it whole and tell the user why. Splitting
surfaces structure that is already there; it doesn't impose structure that isn't.

### Where to cut

Cut the **dependency graph**, not the page count.

- **Cut at the graph's narrow points** — where a linear run forks, or where parallel tracks rejoin.
  Those are the places where "everything before this is finished" is actually true.
- **Give each parallel track its own file.** This is the payoff, not a side effect: two tracks in one
  file are two waves the *same* session runs; two tracks in two files are two sessions that can run
  **at the same time**, on separate branches. Give sibling tracks one number and a letter suffix
  (`M2a`/`M2b`) so the concurrency is visible in the filename, and name in each the one file where a
  merge collision is expected.
- **Keep phases that edit the same core file in the same milestone.** Splitting them hands one
  milestone a half-built version of something another milestone owns.
- **Isolate a phase whose review is one large diff** — a golden-master rebaseline, a mass rename, a
  regenerated fixture. Alone in its own milestone that diff *is* the review, and every gated change
  from the earlier milestones becomes visible in it at once; bundled with other work it's unreadable.
- **1–3 phases per milestone** is the usual landing zone.

### What each milestone file carries

Each milestone must run **standalone** — a session handed only that file should never need to open
another one to start work. Write to `plans-dir/<shared-prefix>-m<N>-<slug>.md` so the family sorts
together in the directory.

- **A header block** placing it in the sequence: `Runs after:` (which milestone must be **merged**,
  not merely started, and why), `Runs beside:` (which milestones may run concurrently), `Unlocks:`,
  and one line on what this milestone delivers.
- **A phase map** — a small table mapping this file's phase numbers to the umbrella's, because phases
  are **renumbered from 1** in each milestone. Without the map, every cross-reference in the umbrella,
  the backlog, and the commit history silently points at the wrong phase.
- **The scoped subset** of Design source, Current state, and Locked decisions — only what binds
  *these* phases. This duplication is deliberate: it's what makes the file standalone, and it's far
  cheaper than a session reading a full survey to find the three paragraphs that bind its work.
- **The phases in full**, plus a dependency diagram and status table covering this milestone only.

### What the umbrella becomes

The original file keeps the **shared context** — the complete Current state survey, the full Locked
decisions register, the design traceability table, and any postmortem or closed-gap analysis — and
**loses every phase**. Add at the top:

- **A banner** stating it is not an executable plan and that `execute-plan` must be pointed at a
  milestone. Without it someone runs the umbrella and gets a plan that declares no phases.
- **The milestone table** — number, link, which of the original phases it covers, what it runs after,
  and status. This table is now where overall progress is tracked.
- **A milestone-level dependency diagram**, and a short **"why the cuts fall here"** note. That note
  is what stops a later session from re-cutting the plan differently or bundling two milestones back
  together because the reason for a seam was never written down.

### Re-point what pointed at the old plan

Splitting an existing plan silently breaks every reference aimed at it — a backlog item whose
`Trigger:` was "the plan's rescale phase lands", a design decision that cited "Phase 7 of
`<plan>`", a sibling plan that sequenced itself after this one. Grep `backlog-dir`,
`design-docs-dir`, and `plans-dir` for the plan's filename and fix what you find. These are
**semantic** rewrites, not find-and-replace: a trigger that fired on a phase now fires on the
milestone that owns it, and phase numbers in the citation are the *umbrella's*, not the milestone's.

Where a reference is both provenance and instruction, keep both halves — cite the milestone so the
reader knows what to act on, and the umbrella so the trail back to the original decision survives.

### The mechanical rule that matters most

**Cross-milestone dependencies never go on a `Depends on:` line.** `execute-plan` parses that line to
build its waves, and it only ever sees the single file it was handed — a `Depends on: Phase 7`
pointing into another milestone is a dependency it cannot resolve, and the phase either gets
scheduled in the wrong wave or blocks forever. State the prerequisite in the header block as prose
instead, and let a milestone's first phase say `Depends on: nothing`, noting alongside it that its
real prerequisite is the previous milestone, merged.

---

## Step 5 — Verify and report

After writing:

1. Re-read the "Current state" and "Locked decisions" sections. Every claim there must be
   verifiable. Remove or mark `[unverified]` anything you're not certain about.
2. Confirm nothing deferred leaked into the plan as an "Out of scope"/"Future work" dumping
   ground — if it did, move it to the backlog per Step 3.5.
3. If a design corpus was used, confirm every rule in the **Design traceability** table maps to a
   phase or a logged deferral, and that you did not silently make a design decision. Anything you
   had to decide about the design *itself* must be surfaced to the user and routed back to
   `design-doc`, not buried in the plan.
4. If you split into milestones (Step 4), check the seams: no `Depends on:` line anywhere names a
   phase outside its own file; every original phase appears in exactly one milestone and in that
   milestone's phase map; the umbrella declares no phases of its own; and each milestone's scoped
   Current state and Locked decisions actually cover the phases in it. A wrong `Depends on:` is the
   failure that surfaces late and costs the most, so check that one by reading the lines, not by
   remembering what you wrote. Then grep for the original plan's filename and confirm no inbound
   reference still points at a phase number the split moved.
5. Tell the user: which plan file was written/updated and how many phases; if you split it, the
   milestone files and what each covers, plus which pairs can run concurrently; which backlog entries
   you added (and to which file); any **design gaps** you found and routed back to `design-doc`;
   and any implementation judgment calls you made (so the user can confirm or override them).

Do not summarize the plan's content back to the user — they can read the file. Just say what
was written and flag your judgment calls.
