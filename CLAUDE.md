# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Lumberjack (the working directory is `CodeInSight`; the shipped plugin, README, and manifests all call it `lumberjack`) is a Claude Code **plugin** — no application code, no build step, no compiled artifacts. It packages four skills in two independent pairs. The **mental-model pair** compares a developer's stated understanding of a codebase against what the code actually does and renders the gaps as graded Mermaid diagrams. The **comment pair** holds the line on comments: one keeps them from being written, the other cuts the ones already there.

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
- `skills/comment-discipline/` — the comment mode; persistent, governs comments as they get written
- `skills/prune-comments/` — the comment review; one-shot report over comments already in the tree
- `skills/<name>/SKILL.md` — frontmatter (`name`, `description`) plus the step-by-step procedure
- `skills/<name>/references/*.md` — progressive-disclosure docs; SKILL.md names which to read at which step, they are not loaded up front
- `skills/<name>/evals/evals.json` — skill-creator eval suite
- `skills/<name>-workspace/` — gitignored; skill-creator scratch space (iteration runs, description-optimization results)

## Architecture

**The mental-model skills are a composed pair.**

- **reality-check** runs standalone: scope the comparison (an area + one of three lenses) → elicit the mental model close to verbatim → investigate the real code (delegated to parallel `Agent` subagents) → grade each claim → build the diagrams → write a report to `docs/mental-model/YYYY-MM-DD-<slug>.md` **in the target repo, not this one**, and never auto-commit it.
- **grounded-plan** wraps reality-check: it runs the reality-check comparison as the grounding phase before Plan Mode, then continues into writing the plan with a Before/After diagram pair. If reality-check is not available as an invocable skill it falls back to following `skills/reality-check/SKILL.md` inline. Both skills ship in this one plugin, so that dependency is normally satisfied.

**Three lenses, each with a fixed diagram form:**
- architecture → Mermaid C4 (`C4Context` / `C4Container` / `C4Component`, chosen by scope)
- deployment → flowchart with subgraphs as host/network boundaries
- data flow → left-to-right flowchart following one or two concrete flows end to end

**Grading is by outline and stroke pattern only, never fill color** — so it survives light and dark rendering and does not collide with color that already carries meaning (team, tech stack). reality-check grades: 🟢 matches / 🟡 partially right / 🔴 contradicted by the code / ⚪ real but unmentioned. grounded-plan's After diagram deliberately uses a *different* palette (new / changing / removed) so a reader who knows both skills does not read "removed" as "wrong". Exact hex values and copy-ready Mermaid snippets live in the `references/` files and must be reused verbatim so reports stay consistent across runs and repos.

**The diagrams are the deliverable; prose stays minimal.** A finding about how two things relate (protocol, push vs poll, call direction) is an *edge* claim, and its correction goes in the edge label phrased as a sentence that reads with its two endpoints — "pushes new scores via Socket.IO, not polled". Misfiling an edge claim onto a node is the specific failure mode both skills exist to avoid. In the findings table, the Evidence column is a bare `file:line` citation — never quoted code, never a sentence.

**The comment skills are a mode/review pair, not a composition.** Neither invokes the other; they split by tense.

**The ladder is defined here and copied into both skills.** Each skill loads on its own, so each has to carry its own operative copy — a shared reference file would not be read (see Editing skills here). This table is the canonical version: the rungs mean the same thing on both sides, and the two skills differ only in tense.

| rung | comment-discipline (before it is written) | prune-comments (already written) |
|---|---|---|
| 1 | Delete it | Delete it |
| 2 | Make the code say it | Make the code say it |
| 3 | Write only the why | Cut it to the why |
| 4 | Write it properly | Keep it |

- **comment-discipline** is persistent and governs comments as code gets written. Rung 1 catches the most: a model's comments are usually artefacts of producing the code — edit scaffolding like `// ...existing code...`, narration of the step just taken, a note addressed to itself.
- **prune-comments** is one-shot and judges comments already in the tree, reported one line per finding, worst first, tagged `lies:` / `commented-out:` / `noise:` / `git-has-it:` / `redundant:`, with `keep:` and `rung N:` as verdict markers in the same slot, ending in a `net:` count. It reports and applies nothing.

Both are modelled on the `ponytail` plugin's mode/review split and sized in its spirit — far shorter than reality-check, with no reference files. **A comment that earns its keep is the thing these skills exist to protect**, not the noise they cut: a why whose reason lives outside the file, a warning, an amplification of a load-bearing line, published API documentation, and machine directives (`# noqa`, `// eslint-disable-next-line`), which are not comments at all and break the build if touched.

## Editing skills here

- Each SKILL.md `description` is tuned for trigger accuracy through a documented iteration process (see `skills/<name>-workspace/description-optimization/`). Treat any change to it as significant and re-run the trigger evals rather than tweaking it casually.
- Prose — both in skill output and in the skill instructions themselves — follows Strunk's *Elements of Style*: terse, active voice, concrete, positive form. See `skills/reality-check/references/writing-style.md`.
- Every diagram edge/arrow label must read as a natural-language sentence that includes its endpoints.
- **Size a new skill like ponytail's — roughly 40–180 lines — and make every section earn its place.** The ceiling is a guideline extrapolated from ponytail's own family (41–120), not a measured law; prune-comments sits above it because it carries a "what earns its keep" list that ponytail-review has no need for, and that list is the part the evals proved load-bearing. Treat a skill drifting past it as a prompt to re-read the sections, not as a failure. A 497-line draft of prune-comments scored no better on its own evals than the 160-line version that replaced it, and its 283-line `references/comment-tags.md` was never opened once across three eval runs. A progressive-disclosure reference file costs real effort and is not read by default — put the content in SKILL.md, or drop it.
- **Instruct a check; naming a category does not produce one.** Cutting that skill lost two findings, and both came back from roughly 13 lines that told the model to go and verify — "Check before cutting: … go and find out why it isn't". The bullets that merely described a category produced no verification at all. Tool-call counts per run track this directly and are the fastest way to spot it: the losing run made 4 calls where the winning ones made 9 and 10.
- **The comment ladder is stated in three places — both SKILL.md files and the Architecture table above — and they move together.** Runtime duplication is forced, because each skill loads alone and a shared reference file would not be read. Silent drift is not: the rungs transposed between the two skills once already, and the Architecture copy went stale during the fix for it. Edit a rung, edit all three, and take the table above as the version that settles disagreements.
- Prefer a harder boundary to a broader description when a skill's output strays into an adjacent domain. "Noting one in passing is fine" licensed whole code-review sections; "at most one closing line — never its own section, never a hunt" stopped it in every run.
