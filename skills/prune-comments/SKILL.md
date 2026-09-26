---
name: prune-comments
description: >
  Reviews the comments already in code and reports which ones to cut, worst
  first: comments that lie about what the code now does, commented-out code,
  changelogs and bylines git already keeps, banner markers, and restatements of
  the line below. Each finding climbs a fix ladder — delete it, make the code
  say it, cut it to the why, or keep it — and stops at the first rung that
  holds. Use whenever someone wants comments reviewed, pruned, cleaned up,
  thinned out, or audited; asks whether the comments still match the code or
  have gone stale; complains about comment noise, redundant javadoc, or
  commented-out code left lying around; or says "prune-comments",
  "/prune-comments", "prune the comments", or "are these comments still true".
  Reports findings and applies nothing. Do NOT use for writing or adding
  comments and docstrings, for general correctness or security review, for prose
  files like READMEs, or for pull-request review comments and discussion
  threads.
---

# Prune Comments

Nothing checks a comment, so comments drift, and one that drifts far enough lies
to the next reader. This pass finds the ones worth cutting and says so in a line
each. It reports; the reader deletes.

A few comments carry what the code genuinely cannot, and cutting those is
vandalism dressed as cleanup. Telling the two apart quickly is the whole job.

## Scope

Whatever the user pointed at — a file, a directory, a diff, the repo. Nothing
named means the current changes: unstaged, then staged, then the last commit.

## Read the code before the comments

Judge each comment against the code it sits on, never against itself. A comment
reads as plausible on its own and gives itself away only next to the function it
describes. Reading the other way round is how a stale comment survives a
cleanup — it sounded reasonable, so it stayed.

This matters most for `lies:`, the most valuable finding and the easiest to
invent. File it only when you can name both halves in one clause: what the
comment claims, what the code does. If you cannot, it is `noise:` or nothing.

The same rule governs a keep. When the reason a comment earns its place lives
outside the file — a server-side default, a generated spec, a lock held
elsewhere — open that file before deciding. A keep taken on plausibility is a
guess wearing a verdict, and this is the expensive direction to get wrong: a
surplus comment costs a reader a second, a deleted warning costs them the bug it
named.

## The ladder

Run every comment down it. Stop at the first rung that holds.

1. **Delete it.** The code says it, or git keeps it. Most comments stop here.
2. **Make the code say it.** Rename, extract, name the constant — the comment
   then has nothing left to say and goes with it.
3. **Cut it to the why.** Keep only what a reader could not recover from the
   code.
4. **Keep it.** Rare. See below.

Delete-first on purpose: rung 1 costs nothing, rung 4 changes nothing. Reaching
for rung 3 when rung 1 holds just rewrites the same lies in better prose.

## Tags

Worst first, and this order is the report's order.

| Tag | The comment | What replaces it |
|---|---|---|
| `lies:` | Contradicts what the code now does | Nothing — name the contradiction |
| `commented-out:` | Dead code nobody dares delete | `git log`. Delete it. |
| `noise:` | Vague, misplaced, or an essay — banner dividers and closing-brace labels too | Nothing, unless a rewrite makes it specific |
| `git-has-it:` | A changelog, a byline, a ticket id recording an edit | `git log`, `git blame` |
| `redundant:` | Restates the line below, or says what the name says | Nothing |

Bulk arrives by copy-paste or by a lint rule. When one habit mints dozens, file
it once with a count and name the rule — forty identical findings is the same
noise problem in a new place.

## What earns its keep

Ask what a reader loses without it. "Nothing, the code says it" means rung 1.

- **A why whose reason lives outside the file** — a library bug with its issue
  link, a spec clause, an ordering that looks arbitrary and is not. A ticket id
  cuts both ways: one recording an edit is `git-has-it:`, one carrying a reason
  stays.
- **A warning about consequences** — not thread-safe, deletes rows.
- **Amplification** — a trivial-looking line that is load-bearing, where the
  next reader's instinct is to tidy it and break something. Check before
  cutting: when a line looks like a redundant round-trip, a pointless copy, or a
  call whose result nobody reads, go and find out why it isn't — a server-side
  default, a mutation guard, a subscription held open. The comment is usually
  the only thing standing between that line and a tidy-up.
- **The unreadable-but-correct** — a regex, a bit-twiddle, a formula from a
  paper.
- **Published API documentation** — docstrings reaching people who never open
  the file. Check before cutting: a docstring that feeds a generated spec or
  client is documentation, and deleting it blanks something downstream. Internal
  docstrings on internal functions get no such pass.
- **Legal and licence headers** the build requires.
- **Machine directives, which are not comments at all** — `# noqa`,
  `// eslint-disable-next-line`, `# type: ignore`, `@ts-expect-error`, shebangs.
  The parser reads these; flagging one breaks the build.
- **`TODO:` and `FIXME:` naming a real deferral**, and `ponytail:` markers.
  Another pass put those there on purpose.

## Output

One line per comment, worst first:

```
<path>:L<n>: <tag> <the comment, quoted short or paraphrased>. <verdict>.
```

Drop the path when the scope is one file. The verdict names the rung concretely
enough to act on without opening the file. The `file:line` is the evidence —
never paste the surrounding code, and check the line number before you cite it.

❌ "Several comments may no longer reflect the current implementation and could
potentially be updated or removed."

✅ `poller.py:L88: lies: "returns once closed". Waits out a blind timeout, then throws. Delete — close_or_timeout() says it.`

✅ `date.py:L12-38: git-has-it: 26-line dated changelog. Delete.`

✅ `cart.py:L140-166: commented-out: the old pricing branch. Delete.`

✅ `user.java:L23: redundant: "@param name the name". 41 more like it — the checkstyle JavadocMethod rule mints them. Drop the rule, then the comments.`

✅ `report.rb:L52: rung 2: "check if the employee is eligible". Extract isEligibleForBenefits(employee); the comment goes with it.`

✅ `parse.c:L201: keep: names the RFC 2045 clause the padding follows.`

End with the count, and nothing after it:

```
net: -<N> comment lines, <M> renames.
```

Nothing to cut: say `Comments earn their keep. Nothing to cut.` and stop. A
review that flags every comment in a file is a mode, not a review, and the
reader stops believing it.

Lead a large scope with the counts, then the worst findings in full. A report
longer than the file it reviews has failed at the thing it is complaining about.

## Boundaries

Comment quality only. A code problem a comment led you to gets at most one
closing line — never its own section, never a hunt. Correctness, security and
performance belong to a normal review pass, and drifting there turns this into a
slower duplicate of one.

This removes comments and never writes them; a request to add them, or to govern
the comments written from here on, is `comment-discipline`'s job, not this
one's. Prose files, and pull-request review
comments, are out of scope. Reports findings, applies nothing. One shot.
