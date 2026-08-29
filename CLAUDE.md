# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Lumberjack (the working directory is `CodeInSight`; the shipped plugin, README, and manifests all call it `lumberjack`) is a Claude Code **plugin** — no application code, no build step, no compiled artifacts. It packages two skills that compare a developer's stated mental model of a codebase against what the code actually does, and render the gaps as graded Mermaid diagrams.

The repo is also its own plugin marketplace: `.claude-plugin/marketplace.json` sets the plugin `source` to `./`.

## Commands

Nothing to build, lint, or unit-test. "Development" means validating the plugin structure and exercising the skills in a live session:

```bash
claude plugin validate .          # check .claude-plugin/ manifests + skill frontmatter
claude --plugin-dir ./            # load this plugin into a session to try the skills
```

Skill authoring and description tuning go through the `skill-creator` skill. Eval suites live at `skills/<name>/evals/evals.json`. The eval **target** is a separate codebase outside this repo at `C:\Users\anton\CodeInSight-eval-targets\AEMS` (a kayaking-competition scoring app) — the evals assume it is checked out there.

## Layout

- `skills/reality-check/` — the diagnostic skill
- `skills/grounded-plan/` — the planning skill; depends on reality-check
- `skills/<name>/SKILL.md` — frontmatter (`name`, `description`) plus the step-by-step procedure
- `skills/<name>/references/*.md` — progressive-disclosure docs; SKILL.md names which to read at which step, they are not loaded up front
- `skills/<name>/evals/evals.json` — skill-creator eval suite
- `skills/<name>-workspace/` — gitignored; skill-creator scratch space (iteration runs, description-optimization results)

## Architecture

**The two skills are a composed pair.**

- **reality-check** runs standalone: scope the comparison (an area + one of three lenses) → elicit the mental model close to verbatim → investigate the real code (delegated to parallel `Agent` subagents) → grade each claim → build the diagrams → write a report to `docs/mental-model/YYYY-MM-DD-<slug>.md` **in the target repo, not this one**, and never auto-commit it.
- **grounded-plan** wraps reality-check: it runs the reality-check comparison as the grounding phase before Plan Mode, then continues into writing the plan with a Before/After diagram pair. If reality-check is not available as an invocable skill it falls back to following `skills/reality-check/SKILL.md` inline. Both skills ship in this one plugin, so that dependency is normally satisfied.

**Three lenses, each with a fixed diagram form:**
- architecture → Mermaid C4 (`C4Context` / `C4Container` / `C4Component`, chosen by scope)
- deployment → flowchart with subgraphs as host/network boundaries
- data flow → left-to-right flowchart following one or two concrete flows end to end

**Grading is by outline and stroke pattern only, never fill color** — so it survives light and dark rendering and does not collide with color that already carries meaning (team, tech stack). reality-check grades: 🟢 matches / 🟡 partially right / 🔴 contradicted by the code / ⚪ real but unmentioned. grounded-plan's After diagram deliberately uses a *different* palette (new / changing / removed) so a reader who knows both skills does not read "removed" as "wrong". Exact hex values and copy-ready Mermaid snippets live in the `references/` files and must be reused verbatim so reports stay consistent across runs and repos.

**The diagrams are the deliverable; prose stays minimal.** A finding about how two things relate (protocol, push vs poll, call direction) is an *edge* claim, and its correction goes in the edge label phrased as a sentence that reads with its two endpoints — "pushes new scores via Socket.IO, not polled". Misfiling an edge claim onto a node is the specific failure mode both skills exist to avoid. In the findings table, the Evidence column is a bare `file:line` citation — never quoted code, never a sentence.

## Editing skills here

- Each SKILL.md `description` is tuned for trigger accuracy through a documented iteration process (see `skills/<name>-workspace/description-optimization/`). Treat any change to it as significant and re-run the trigger evals rather than tweaking it casually.
- Prose — both in skill output and in the skill instructions themselves — follows Strunk's *Elements of Style*: terse, active voice, concrete, positive form. See `skills/reality-check/references/writing-style.md`.
- Every diagram edge/arrow label must read as a natural-language sentence that includes its endpoints.
