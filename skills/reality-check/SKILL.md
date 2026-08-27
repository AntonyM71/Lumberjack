---
name: reality-check
description: >
  Compares what a developer currently believes (or what an existing doc/wiki claims) about a codebase area — architecture, deployment, or data flow — against what the code actually does today, then writes a graded Mermaid comparison report (a "your model" diagram next to a red/amber/green-outlined "reality" diagram). Use this whenever someone wants to check whether their understanding of a system's structure is still accurate, wants a real, doc-independent walkthrough of an unfamiliar service before touching it (especially when they say they don't trust the existing wiki/docs), or is grounding a refactor/incident/planning discussion in what's actually true. This includes a quick one-line question like "does X still use Y, or did that change?", not only long walkthroughs. Trigger on requests like "how does X actually work" (about a service/system's structure, not a business rule or pricing/calculation logic), "am I still right about how this works", "walk me through the architecture/deployment/data flow of X" (including "I just inherited this and don't trust the docs"), "what changed in this system since I last looked", or "before we redesign this, let's make sure I understand it" — even if the user never says "mental model" explicitly, and even when framed around debugging an incident, as long as it references a specific belief about how the flow works and asks to verify it. Also trigger before planning or brainstorming work on an existing system when confirming a stated belief would ground the discussion — not for a bare "propose a design/migration plan" request with no current understanding to check first. Skip it for business-logic/pricing-calculation questions, code review, a from-scratch diagram/doc request from someone who has never looked at the repo and isn't checking anything against reality, and plain "here's an error, fix it" debugging with no belief about how things are supposed to work.
---

# Reality Check

Mental models of a codebase drift from the code the moment someone stops
looking at it — a service gets split, a library gets swapped, a migration
happens per some ADR nobody re-read. This skill makes the drift visible
instead of assumed. It compares what a specific person believes about a
specific area against what the code currently does, and hands back a report
built around diagrams, not prose, so the reader can spot the gaps in seconds.

The output is one Markdown file with two Mermaid diagrams: the model as
described, and reality with each part graded on how well it matched. Keep
prose everywhere else minimal — bullets and short captions, never an essay.
The diagrams are the deliverable.

## Flow

1. Scope the comparison (area + lens)
2. Elicit the mental model
3. Investigate reality
4. Grade the model against reality
5. Build the two diagrams
6. Write the report and summarize concisely

### 1. Scope the comparison

You need two things before asking anything else: **what area** of the system,
and **what lens** to view it through. If the user's request already gives
both (e.g. "how does deployment actually work"), don't ask again — restating
back a scope they already gave reads as not having listened.

The three lenses, and when each fits:

- **Architecture (C4-style)** — components, services, modules, and how they
  call each other. Pick the C4 fidelity to match the area's size: a single
  module or feature → Component diagram; one service → Container diagram;
  multiple services / "the whole system" → Context diagram. Don't default to
  Context for everything — a Component-level diagram of one feature is more
  useful than a Context diagram so zoomed out it says nothing.
- **Deployment** — what actually runs where at runtime: processes,
  containers, networks, managed services, ports. This is about the runtime
  topology, not the source tree.
- **Data flow** — where data enters, how it's transformed/stored along the
  way, and where it exits or ends up. Follow one or two concrete flows
  end-to-end rather than every possible path.

If the area or lens is genuinely ambiguous, ask exactly one question, e.g.:
"Which part, and are you checking the architecture, how it's deployed, or
how data moves through it?" Don't ask about both if only one is missing.

If a request doesn't cleanly fit one lens (e.g. "how does auth work" touches
both architecture and data flow), say which lens you're leading with and
why, rather than cramming everything into one crowded diagram. A second,
smaller diagram is better than one that tries to show everything at once.

### 2. Elicit the mental model

If the user already described their understanding in the initial request —
many will front-load it rather than wait to be asked — use that as-is and
move on; asking them to repeat what they just told you reads as not having
read the message. Only ask when it's genuinely missing.

When you do need to ask, invite roughness — "sketchy or half-remembered is
fine, that's exactly what we're checking" — because a hedged, over-careful
answer defeats the point: you're comparing their actual belief, not a
version they've double-checked before telling you.

Don't lead. Don't hint at the right answer, correct them as they talk, or
ask questions shaped like the answer ("...and it uses Postgres, right?").
If their first answer is thin (a sentence or two), ask 2-3 follow-ups suited
to the chosen lens to get enough substance to actually compare — for
example:

- Architecture: "What are the main pieces, and which ones talk to which?"
- Deployment: "What processes/containers exist, and what talks to what over
  the network?"
- Data flow: "Where does the data come in, what happens to it, and where
  does it end up?"

Record what they say close to verbatim — this becomes the "your model"
diagram. Silently tidying it up while transcribing erases the exact
mismatches you're trying to surface.

### 3. Investigate reality

For anything beyond a couple of files, delegate exploration to the `Agent`
tool (Explore or general-purpose) rather than reading everything inline —
it keeps your own context clean for the comparison step and covers more
ground in parallel. Spin up more than one agent when the area spans
multiple services or concerns (e.g. one agent per service for an
architecture lens, or one per hop for a data-flow lens).

What to look for depends on the lens:

- **Architecture**: entry points, module/service boundaries, and how they
  actually communicate (imports, DI wiring, HTTP calls, queues, sockets) —
  not just what a folder is named.
- **Deployment**: Dockerfiles, compose files, k8s manifests, IaC, CI/CD
  pipeline definitions, `.env`/config files. The runtime topology, including
  things easy to miss from source alone: networks, health checks, exposed
  ports, restart policies, external dependencies.
- **Data flow**: entry points (API routes, forms, webhooks, uploads),
  transform/storage steps (DB, cache, queue, background jobs), and exit
  points (responses, exports, downstream events, UI updates).

Treat existing docs, diagrams, and CLAUDE.md/README architecture notes as a
*hypothesis*, not ground truth — documentation drifts too, sometimes in the
opposite direction from the user's memory. Verify against the code itself
before trusting a comment or doc that claims something.

### 4. Grade the model against reality

Break the elicited mental model into distinct claims — one per component,
connection, or step the user described. Grade each against what you found:

| Grade | Meaning |
|---|---|
| 🟢 Green | Matches reality |
| 🟡 Amber | Partially right — correct idea, wrong detail, outdated, or incomplete |
| 🔴 Red | Wrong — contradicted by the code |
| ⚪ Grey | Real, but the user never mentioned it at all (a gap, not a wrong belief) |

Grey is not "wrong" — it's an unknown-unknown, and worth listing separately
from the red/amber/green claims since it's a different kind of finding (a
missing part of the model, not an incorrect part of it).

**Decide early whether each claim belongs on a node or an edge**, because
that decision determines how it gets shown in step 5. A claim about a
*component* — it exists, it does X, it's built with Y — is a node claim.
A claim about *how two things relate* — which protocol, push vs poll, sync
vs async, which direction a call goes — is an edge claim. "We use plain
WebSockets" and "the display polls every few seconds" are both edge claims:
they're wrong about a connection, not about whether a component exists.
Misfiling an edge claim as a node claim is exactly what pushes a finding
out of the diagram and into prose, which is the failure mode this skill
exists to avoid — the diagram should be able to state "pushed via
Socket.IO, not polled" directly on the arrow, not just at the two
components it connects.

For every grade, note **one concrete piece of evidence** — a file path
(ideally file:line), config key, or specific behavior — never a vague "this
seems off." If you can't point to the specific evidence, you haven't
investigated enough yet; go back to step 3 rather than guessing at a grade.
Keep the evidence itself terse (see step 6) — the rigor is in having a real
citation, not in how much you write about it.

### 5. Build the two diagrams

Two Mermaid diagrams, matched to the lens chosen in step 1:

1. **Your model** — a plain, ungraded rendering of exactly what the user
   described. No colors, no judgment. This is a mirror, not a verdict.
2. **Reality** — the actual structure, with every node/relationship styled
   by its grade from step 4, plus a legend.

Read [references/mermaid-patterns.md](references/mermaid-patterns.md) before
writing the reality diagram — it has copy-ready snippets for C4 element
styling, flowchart grade outlines, the legend block, and a
deployment/data-flow subgraph pattern. Mermaid's syntax is picky about
exact placement of style/class statements; reusing the templates avoids
diagrams that fail to render. Grade with **outlines/borders only** — don't
recolor fills — so the diagram still reads cleanly in both light and dark
rendering and the grade doesn't fight with any other meaning color might
carry (e.g. a service already colored by team).

### 6. Write the report and summarize concisely

Save the report inside the repo being analyzed at:

```
docs/mental-model/YYYY-MM-DD-<topic-slug>.md
```

This mirrors how specs and plans get saved elsewhere in a Claude Code
planning workflow (dated, topic-named, under `docs/`) so a later planning
step can find it by convention. Use the exact template below. Don't commit
it automatically — this is a working diagnostic artifact, not a decision
record, and running this skill repeatedly would clutter history with
auto-commits. Mention the path and let the user decide whether to keep it
under version control.

```markdown
# Reality Check: <topic>

**Lens:** <architecture | deployment | data flow> · **Date:** <date> · **Scope:** <area>

## TL;DR
- <biggest red first — the thing most worth knowing>
- <next biggest gap>
- <one line on what mostly held up, if anything>

## Your model

<1-2 sentence restatement of what the user described, no commentary>

\`\`\`mermaid
<ungraded diagram of the stated model>
\`\`\`

## Reality

\`\`\`mermaid
<graded diagram, styled per references/mermaid-patterns.md, legend included>
\`\`\`

## What's off, and why

| Grade | Claim | Evidence |
|---|---|---|
| 🔴 | <specific claim> | <file:line — short pointer> |
| 🟡 | <specific claim> | <file:line — short pointer> |
| ⚪ | <thing not in their model at all> | <file:line — short pointer> |
```

Only list rows worth flagging — skip green claims unless a green result is
itself surprising or worth confirming explicitly (e.g. "yes, still true
despite the big refactor last month").

The Evidence column is a **citation, not a second explanation**. One line:
a file path (ideally `file:line`) plus a handful of words of paraphrase —
enough for the reader to go verify it themselves. Never quote code blocks,
stack multiple citations, or write multi-sentence justifications in this
table; if a finding needs more than that to land, that's a sign the finding
itself belongs on the diagram (a node label, or an edge label per
references/mermaid-patterns.md) rather than being explained here. The
diagram is what the reader looks at first — the table should never be doing
work the diagram could do instead.

In chat, don't restate the report. Point to the file path and give a 3-5
bullet TL;DR led with the reddest finding, then stop — the diagrams do the
explaining once the user opens the file.

## Boundaries

This skill produces a diagnostic comparison, not a design proposal. If the
conversation moves from "is my understanding correct" to "here's how I want
to change it," that's a different kind of work (design/planning) — hand off
rather than blending a redesign into the comparison report.
