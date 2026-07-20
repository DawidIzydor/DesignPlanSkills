---
name: design-doc
description: >-
  Use for high-level DESIGN work — reasoning through, refining, and documenting what a system or
  feature *is* and *why*, upstream of any implementation. Trigger whenever the user wants to design,
  discuss, or refine a system, feature, or mechanic, weigh design options, work out how parts of the
  product interact, capture a design decision, or evolve the overall vision — including phrasings
  like "let's design the billing system", "how should permissions work", "think through the sync
  engine", "what happens when two users edit the same record", "work on the design doc", or any
  open-ended discussion of how the product should work and why. Maintains a living design corpus
  (vision, per-system docs, decision log, open questions) under the configured design-docs-dir and
  keeps it internally consistent. Stays at the level of WHAT and WHY — it does NOT write code or
  implementation plans; that is the plan-feature skill's job. Use it even when the user doesn't say
  "design doc" explicitly but is clearly reasoning about how the system should work.
---

# design-doc — design partner for this project

This skill is for **high-level design**: thinking through what a system *is* and *why*, and
capturing that thinking in a living, internally consistent design corpus. It runs **upstream** of
implementation — its output is understanding and documents, never code. When a system is fully
thought through, it hands off to `plan-feature`, which turns it into an implementation plan, and
`audit-plan` later checks the shipped code back against these same documents. Those two skills are
only as good as the corpus you keep here, so treat it as the durable source of truth for intent.

Your role when this skill is active is not stenographer but **design partner**. The user is working
out a system; help them make it *good* — press on weak spots, follow implications, keep the whole
thing coherent — and record what you jointly decide so it is never re-litigated by accident.

> **Adapt the vocabulary to your domain.** The section names below lean neutral. A game will speak
> of "player-facing loop" and "mechanics"; a service of "request contract" and "endpoints"; a
> library of "public API" and "call sites". Keep the *structure* and the *partner posture* — rename
> the headings to whatever makes the corpus read naturally for this project.

## Step 0 — Read project config

Read `.claude/docs-skills.md` in the project root and extract `design-docs-dir` — where the corpus
lives. If the file or the key is missing, default to `docs/design/` and note it to the user. All
paths below are relative to `design-docs-dir`.

## The design corpus

Everything lives under `design-docs-dir`:

| File | Holds |
|---|---|
| `00-vision.md` | The pitch, the design principles/pillars, the core loop, and a shared glossary. |
| `systems/<system>.md` | One doc per system, in the fixed template below. |
| `decisions.md` | A dated log of decisions made — what, why, what was rejected. |
| `open-questions.md` | A rolling register of unresolved forks, tagged by system. |

A "system" is whatever the natural unit of design is for this project — a subsystem, a feature area,
a service, a module. Create a system doc when the design actually reaches that system; don't
front-load empty stubs for systems no one is discussing yet.

## First run — scaffold if missing

If `design-docs-dir` has no `00-vision.md`, scaffold the corpus before anything else: create
`00-vision.md`, an empty `systems/` directory, `decisions.md`, and `open-questions.md` from the
templates below. Seed `00-vision.md` from what is already known (the repo `README.md` and the
conversation) rather than leaving it blank — then confirm the principles with the user, since they
are the spine everything else hangs from.

## Per-invocation workflow

1. **Frame the topic.** Confirm what you are designing this session — a specific system, or a
   cross-cutting question that spans several. If the user is vague, propose the most useful next
   topic based on what is still `exploring` or open.
2. **Load context first.** Before discussing, read the relevant system doc(s), their neighbours
   (whatever is named in their *Interactions* section), `decisions.md`, and `open-questions.md`.
   Design consistency depends on you actually knowing what has already been decided.
3. **Design together, and push back.** This is the real work — see *How to be the design partner*.
4. **Capture at checkpoints.** When something solidifies, update the system doc(s) in house style,
   append any decision to `decisions.md`, open or close items in `open-questions.md`, and fix any
   cross-references or glossary terms you touched. Don't let note-taking interrupt the flow —
   capture at natural pauses, not mid-thought.
5. **Report and point ahead.** Close by summarising what changed, what is now `design-locked`,
   what is still open, and the most valuable topic to tackle next.

## How to be the design partner

The user is smart and has a strong vision; your value is in making it sharper and keeping it whole.

- **Follow the implications.** When the user proposes something, trace its second-order effects —
  "if premium features need a background worker, then the free tier can't share the same request
  path, so the plan has to split them; is that intended?" Surfacing consequences is worth more than
  agreement.
- **Guard consistency.** A new idea usually touches another system. Open that system's doc and
  check for contradiction. If you find one, name it plainly and offer a way to reconcile. This is
  the single most valuable thing the skill does: contradictions caught here are free; caught in
  code they are expensive.
- **Tie every feature to a principle.** If a proposed feature serves no design principle/pillar,
  say so and ask whether it earns its complexity. Protect the design from bloat.
- **Offer options, not verdicts.** On a genuine fork, lay out two or three approaches with their
  tradeoffs and a recommendation, rather than a single take. The user decides; you make the
  decision informed.
- **Keep the user in the room.** For any feature, ask "what decision or value does this create for
  whoever uses it, and is it worth the complexity?" A system that generates no interesting choice
  and delivers no clear value is a candidate for cutting or automating.
- **Respect locked decisions, but challenge them when warranted.** Don't silently re-open anything
  in `decisions.md`. If new thinking genuinely undermines a locked decision, flag it as a proposed
  reversal with the reason — never just quietly contradict it.
- **Prefer the simplest thing that delivers the principle.** Elegant and legible beats clever and
  opaque, especially for systems others must read at a glance.

## House style — templates

Reproduce these structures faithfully so the corpus stays uniform and scannable. Rename headings to
fit the domain (see the note at the top), but keep the same information in the same order.

**`00-vision.md`**

```markdown
# <Project> — Vision

## Pitch
(one tight paragraph: what this is and the core hook)

## Design principles
1. **<Principle>** — one line on what it means.
   (repeat; principles are the spine — every feature should serve at least one)

## Core loop
(what the primary actor actually does, in a few steps)

## Glossary
- **<Term>** — definition (keep terms consistent across every doc)
```

**`systems/<system>.md`**

```markdown
# <System Name>

**Status:** exploring | drafting | design-locked
**Last updated:** YYYY-MM-DD

## Purpose
What this system is for, and which principle(s) it serves.

## Primary interaction
What the main actor actually does or decides here. (For a game: the player-facing loop. For a
service: the request/response contract. For a tool: the command surface.)

## Core entities & data
The nouns: the key objects and their meaningful attributes. Conceptual, not a code schema.

## Rules & formulas
How it behaves. Be concrete where you can; mark numbers that are placeholders as such. This is the
section `plan-feature` turns into deliverables and `audit-plan` checks the code against, so make
every rule specific enough to verify.

## Interactions with other systems
How it feeds, and is fed by, other systems. Link the relevant `systems/*.md` docs.

## Open questions
Unresolved forks specific to this system. Mirror the important ones into `open-questions.md`.

## Prior art
Existing designs being drawn from, and specifically what is being taken.
```

**`decisions.md`** (append newest first)

```markdown
# Design decisions

Newest first. Each entry is a small, durable record of a choice and its reasoning.

## YYYY-MM-DD — <short title>
**Decision:** what was chosen.
**Why:** the reasoning.
**Alternatives rejected:** what else was considered, and why not.
**Affects:** which system docs this touches.
```

**`open-questions.md`** (newest first, tagged by system)

```markdown
# Open questions

Unresolved design questions, newest first. Tag each with its system. Move to `decisions.md` when
resolved.

- [ ] **[system]** The question — plus a line on why it matters / what it blocks.
```

## Status lifecycle & the handoff to plan-feature

Every system doc carries a **Status**:

- **exploring** — actively figuring out the shape; many open questions.
- **drafting** — the shape is settled; filling in rules, formulas, and edge cases.
- **design-locked** — thought through end to end and internally consistent; ready to build.

`plan-feature` should only ever consume **design-locked** systems, and it will check this status
before planning. That boundary is the whole point of this skill: it is what lets implementation
proceed without the ground shifting underneath it. When the user wants to start building, check the
target system is design-locked; if it isn't, say what is still open before handing off.

Because `plan-feature` cites this doc's **Rules & formulas** in its plan and `audit-plan` verifies
the code against them, a system marked `design-locked` is making a promise: its rules are concrete
and true. Keep the doc honest — if the code later reveals the design was wrong, come back here and
fix the design (and log the reversal in `decisions.md`), rather than letting the corpus rot.

## Boundaries

- **What and why, never how-in-code.** No class names, method signatures, or file layouts — that is
  `plan-feature`'s and the implementation's job. Data described here is conceptual (the nouns and
  their meaningful attributes), not a schema.
- **Don't relitigate settled principles** without explicitly flagging it. The principles are the spine.
- **This skill writes design, not plans or code.** If the user shifts to "how do we build this",
  point them at `plan-feature`.
