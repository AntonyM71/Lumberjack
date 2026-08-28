---
name: grounded-plan
description: >
  Plans a change to a codebase's architecture, deployment, or data flow by first grounding it in how the relevant part of the system actually works today, then bakes a before/after diagram pair into the implementation plan so the change is visibly justified rather than just described in prose. Use this whenever someone wants to plan a feature, refactor, migration, or fix on a system they already hold beliefs about — e.g. "I want to add X, and I think Y currently works like Z" or "let's build this on top of the queue, which I think still runs through RabbitMQ" — even if they never ask to be "grounded" explicitly. Also use it for a brand-new, "first-of-its-kind" service or component being built for real — even one the requester describes as having nothing existing to build on almost certainly still has to talk to the existing message bus, auth system, or deploy pipeline, and that assumption of isolation is itself worth checking, not taken at face value. Elicit the current understanding of whatever the plan depends on, compare it against the actual code using the reality-check skill, and let what that surfaces shape the plan before it's written. This composes with — not instead of — Claude's normal decision to enter Plan Mode for non-trivial implementation work. Don't trigger for a throwaway prototype or spike that isn't meant to become part of the real system at all (plan normally instead), for a belief about business/pricing/calculation logic rather than system structure (reality-check's comparison doesn't cover that domain either), or for a request that only asks whether an existing belief is accurate with no intent to build anything — that's reality-check alone, hand off to it.
---

# Grounded Plan

Plans built on a stale picture of the system inherit its mistakes silently —
a step that assumes the wrong protocol, an estimate that ignores a service
nobody mentioned, a design that "extends" something that was already
replaced. This skill fixes the order of operations: check the belief the
plan would otherwise be built on, surface where it's wrong, and only then
write the plan — with a diagram of where things stand today next to a
diagram of where the plan takes them, so the change is visibly justified
rather than asserted in prose.

This is [reality-check](../reality-check/SKILL.md)'s sibling, not a
replacement for it: reality-check answers "is my understanding still
right?" on its own; this skill asks the same question in service of a
plan, then continues into writing that plan. See Dependencies below.

## Flow

1. Confirm this is the right tool for the request
2. Elicit the current understanding of the relevant area
3. Compare it against reality (via reality-check)
4. Present the gaps and refine the intended change together
5. Enter Plan Mode and write the plan file
6. Exit Plan Mode for approval

### 1. Confirm this is the right tool

Two things need to both be true, or this skill is the wrong fit:

- **There's build intent, and it's about the codebase's structure, not its
  business logic.** The user wants to add, change, remove, or migrate
  something — not just check whether they understand the system correctly.
  If there's no build intent at all, hand off to reality-check alone; don't
  fold a pure diagnostic request into a plan nobody asked for. If the
  belief is about business/pricing/calculation rules rather than
  architecture, deployment, or data flow, this skill is also the wrong
  fit — reality-check's comparison step doesn't cover that domain, so
  there's nothing for this skill to delegate to.
- **There's something real for the plan to depend on being right about.**
  Usually that's the thing being changed directly — it already exists and
  already works some way, and the user has (or can be asked for) an
  assumption about how it currently behaves. A brand-new component still
  counts, and counts more often than it might seem: a "first-of-its-kind"
  service being built for real still has to talk to the message bus,
  authenticate through the existing system, or deploy through the existing
  pipeline, and a wrong belief about any of those is exactly as
  plan-derailing as a wrong belief about the component being modified
  directly — don't take "there's nothing existing to build on" at face
  value just because the user framed it that way; that belief about the
  new work's isolation is itself worth checking. The genuine exemption is
  narrower than "brand new": a throwaway prototype or spike that isn't
  meant to become part of the real system at all, where nothing existing
  constrains the design because nothing existing is actually going to meet
  it.

When both hold, this skill's grounding phase runs as part of the normal
exploration Claude already does before planning — it's not an extra gate
the user has to ask for separately.

### 2. Elicit the current understanding

Same spirit as reality-check's own elicitation step: if the user already
stated their belief about the relevant area in their request, use it as-is.
If it's thin or missing, ask — inviting roughness ("rough or half-remembered
is fine") rather than a hedged, double-checked answer, since a hedged
answer defeats the point of checking it. Scope the question to the area the
plan actually touches, not the whole system.

### 3. Compare it against reality

Delegate this step to the **reality-check** skill rather than
re-implementing its investigation-and-grading logic here — it already knows
how to pick a lens (architecture, deployment, or data flow), delegate
investigation to subagents, and grade claims red/amber/green/grey with
concrete evidence. If reality-check is available as a skill in this
session, invoke it. If it isn't (e.g. this skill's own instructions are
being followed directly in an environment where cross-skill invocation
isn't wired up), read `../reality-check/SKILL.md` and follow its steps
2 through 4 inline instead — same grading table, same
[mermaid-patterns.md](../reality-check/references/mermaid-patterns.md)
styling.

The output of this step is what reality-check normally produces: an
ungraded "your model" diagram, a graded "reality" diagram, and a small
standalone red/amber/green/grey legend beside it.

**Investigate as thoroughly here as reality-check would running on its
own — don't let the fact that a plan follows shrink this step.** It's
tempting to treat this as a quick check on the way to the real work, but
the plan is only as good as this investigation: a shallow pass here
produces a plan built on the same size of blind spot the belief-check was
supposed to remove, just moved one step later. Delegate to multiple
parallel subagents the way reality-check's own step 3 does (one per
service/area/hop) rather than doing a couple of direct reads yourself and
calling it done.

This step is also where the plan's real risks and constraints tend to
surface, not just whether the one stated claim was right — existing docs
that mention a scaling limit, a security posture, a rate limit, an
inconsistency between two related pieces of config, anything that the
*new* work would touch or make worse. reality-check on its own has no
reason to go looking for this (it's only checking a belief against
reality), but a plan built without it misses exactly the things that turn
into incidents. Read what reality-check's own investigation turns up with
an eye toward "does this matter to what's about to be built," not only
"was the claim true."

**Once the stated belief turns out wrong, keep asking whether the original
approach would have been a bad idea anyway, independent of that specific
correction.** It's easy to stop at the first disqualifying fact — "there's
no polling loop, so scratch that" — and move straight to the corrected
design. But if the original approach had other problems that have nothing
to do with the factual error (a frontend webhook call would leak a
credential to anyone with dev tools open regardless of whether it's
triggered by a poll or a push, say), a plan that only states the one
factual correction leaves the door open for someone to "fix" the wrong
detail and reintroduce the actual problem — e.g. "let's add a polling loop
then." Surfacing every independent reason, not just whichever one happens
to be the factual correction, is what makes the corrected approach the
obviously right one rather than a workaround for one specific complaint.

### 4. Present the gaps and refine the plan

Show the comparison from step 3 to the user before writing anything into a
plan file — this is a conversation checkpoint, not a silent internal check.
Lead with whatever's reddest, the same way reality-check's own summary
does. Then ask how it changes what they want to build: a wrong protocol
belief might change an integration approach; an unmentioned service might
become a dependency the plan needs to account for; a component that turned
out not to exist might mean the whole approach needs rethinking.

Let the user's answer actually reshape the plan's scope. Don't treat this
as a formality on the way to a plan you'd already decided on.

### 5. Enter Plan Mode and write the plan file

Once the change is grounded and refined, continue into Claude Code's normal
Plan Mode if not already in it. Write the plan to the file the harness
provides — this skill does not invent its own doc location or path
convention the way reality-check does, because a grounded plan is meant to
flow into the same approve-then-implement cycle as any other plan.

Use this structure in the plan file:

```markdown
# <Plan title>

## Grounding
<2-4 bullets: the specific corrections from step 3 that shaped this plan.
Not the full comparison table again — the user already saw that in step 4.
Only include a correction here if it actually changed something about the
plan below; drop anything that turned out not to matter.>

## Before

\`\`\`mermaid
<plain, ungraded diagram of the system as it actually is today, per
step 3's reality diagram — same lens, same diagram type, but with the
red/amber/green grading removed. It served its purpose finding the gaps in
step 4; by the time this is written it's just an accurate diagram, and
grading marks with nothing left to grade against would confuse a reader
who never saw the "your model" diagram this plan doesn't keep.>
\`\`\`

## After

\`\`\`mermaid
<the same diagram with the plan's changes applied, new/changed/removed
elements marked per references/plan-diagram-patterns.md. This is the
"why" for the steps below made visible — a reader should be able to spot
what's actually changing from this diagram alone before reading a word of
the implementation steps.>
\`\`\`

## Implementation steps
<normal technical plan content — the parts of a plan that would exist
regardless of whether this skill ran: files to change, order of operations,
risks, testing approach.>
```

Read
[references/plan-diagram-patterns.md](references/plan-diagram-patterns.md)
before drawing the After diagram — it covers marking new/changed/removed
elements in both C4 and flowchart diagrams, reusing the mechanism
mermaid-patterns.md repurposes for grading, but for its original purpose
here: highlighting a delta.

Where a piece of the implementation has a specific, checkable shape — a new
compose service definition, a function signature, a payload schema, an env
var and its default — write the actual snippet, not just a sentence
describing that one will exist. "Add a redis service to docker-compose.yaml"
leaves the reader to invent the exact image, healthcheck, and network
settings themselves; a five-line YAML block that already reflects what step
3 found (no host port, no volume, default network only) is something they
can actually check against what gets built, and it's barely more effort to
write once the investigation has already settled those details.

### 6. Exit Plan Mode for approval

Use `ExitPlanMode` once the plan file is complete, same as any other
planning task. Implementation then proceeds the normal way once approved —
nothing about this skill changes what happens after approval.

## Why the "your model" diagram doesn't survive into the plan

Three diagrams exist over the course of this flow, but only two end up in
the plan file:

1. **Your model** (step 3) — a tool for finding the gaps. Once step 4 is
   done, it's served its purpose.
2. **Reality, graded** (step 3) — also a tool for that same conversation;
   the grading marks are meaningless to someone who never saw diagram 1.
3. **Reality, plain** → **Before** in the plan file — the same structure as
   diagram 2, with the grading stripped, since by now it's just true.
4. **After** in the plan file — diagram 3 with the plan's changes marked.

If the drift between belief and reality was large enough that it's worth a
permanent record independent of this plan, that's what a standalone
reality-check report is for — run it separately rather than keeping the
graded diagram around here as a substitute.

## Dependencies

This skill assumes **reality-check** is installed alongside it (both ship
in the same plugin). If a request would trigger this skill but
reality-check isn't available, fall back to doing the comparison inline
per step 3's fallback instructions rather than skipping grounding
entirely — the point of this skill is the grounding, not the delegation
mechanism.

## Boundaries

- No build intent → reality-check alone, not this skill.
- No existing belief to check (pure greenfield) → plan normally, not this
  skill.
- If the conversation starts here but the user decides mid-way they just
  want the comparison and not a plan right now, that's fine — finish steps
  1-4, hand them the comparison, and stop rather than pushing into a plan
  they didn't ask for.
