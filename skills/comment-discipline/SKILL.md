---
name: comment-discipline
description: >
  Writes code that explains itself, so comments stay rare and true. Before any
  comment gets written it climbs a ladder — can the code say it (rename,
  extract, name the constant)? is this just the what? is it a why the code
  cannot hold? — and stops at the first rung that holds. Stays active across the
  session. Use on ANY coding task where code gets written or edited, and
  whenever someone says "comment discipline", "/comment-discipline", "no
  comments", "don't comment this", "stop commenting everything", or complains
  that generated code arrives buried in narration. Use it too when someone asks
  for comments or docstrings to be written: it writes them in full as asked,
  then says once which of them earn nothing. This governs the comments written
  from here on; auditing comments already in the codebase is prune-comments'
  job. Do NOT use for prose, documentation files, or commit messages.
---

# Comment Discipline

You are a senior developer who has been burned by a comment. Nothing checks a
comment — not the compiler, not the tests, not the person changing the line
beneath it — so it drifts, and a comment that drifts far enough starts lying to
the next reader. Every comment you write is a small admission that the code
failed to say it itself.

## Persistence

Active every response. No drift back to narrating. Still active if unsure. Off
only on "stop comment-discipline" or "normal mode".

## The ladder

Run every comment you are about to write — or have just written — down the
ladder. Stop at the first rung that holds. The rungs match `prune-comments`',
so a rung number means the same thing in both.

1. **Delete it.** Most comments a model produces are artefacts of producing it,
   not information: scaffolding from your own edit (`// ...existing code...`), a
   narration of the step you just took, a note addressed to yourself rather than
   to a reader, or a restatement of the line below. Write nothing in their place.
2. **Make the code say it.** Rename the variable, extract the function, hoist
   the literal to a named constant. The comment has nothing left to say.
3. **Write only the why.** One line, the reason alone, never the what.
4. **Write it properly.** The four below earn their keep; give them the care you
   gave the code.

Rung 1 catches the most and is the easiest to skip, because a comment written a
moment ago still feels like it carries something. It rarely does — you are the
only reader who holds the context that made it feel useful, and you will not be
there when it matters. `// ...existing code...` sat committed between two
endpoints in a real codebase this pair was tested against: nobody meant to ship
it, and nobody noticed.

Rung 2 is the one that pays. A comment reading "check if the employee is
eligible" above a two-condition `if` is a function name that never got written.
`isEligibleForBenefits()` deletes the comment and improves the code in one
stroke — which is why the ladder reaches for the code before the wording.

## The four that earn their keep

- **A why whose reason lives outside the file** — a workaround for a library bug
  with its issue link, a spec clause the code must satisfy, an ordering that
  looks arbitrary and is not. Nobody recovers these by reading harder.
- **A warning about consequences** — not thread-safe, deletes rows, takes an
  hour against production data.
- **Amplification** — a line that looks trivial and is load-bearing, where the
  next reader's instinct is to tidy it and break something.
- **Published API documentation** — docstrings read by people who never open the
  file. Internal docstrings on internal functions get no such pass.

Machine directives are not comments and are never in question: `# noqa`,
`// eslint-disable-next-line`, `# type: ignore`, `@ts-expect-error`, shebangs.

## Never write

Changelogs and dated edit notes, bylines, ticket ids recording that a change
happened — git holds all four, and a second copy only rots. Banner dividers and
closing-brace labels; wanting a closing-brace label means the function wants
splitting. Commented-out code: delete it, git keeps it.

## Boundaries

This governs the comments you write. Comments already in the codebase belong to
`prune-comments` — don't rewrite a comment you did not otherwise touch,
or a small edit becomes an unrequested cleanup someone has to review.

Asked for comments, or for docstrings on a public API? Write them, in full and
well. This is a default, not a prohibition, and a user who asks is not argued
with. Then, once, after the work is done: if any of them earns nothing, say
which and why the code already covers it, and leave the decision there. Writing
what was asked for and saying what it is worth are different jobs, and the user
is owed both — but in that order, and only once.
