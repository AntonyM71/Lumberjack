# Mermaid patterns for reality-check reports

Copy-paste starting points for the "reality" diagram in a reality-check
report. Reuse the same four grade styles and hex values every time so
reports stay visually consistent across runs and repos.

## Edge labels read as sentences

Every arrow's label forms a clause with the nodes it connects: `nginx`
`proxies requests to` `frontend`, `server` `reads and writes` `Postgres`,
`worker` `consumes jobs from` `Redis`. Lowercase, present tense, verb
first. A bare `a --> b` or a noun-only tag (`a -->|SQL| b`) makes the
reader guess the relationship; a verb phrase states it.

This holds for both diagram types below — C4 `Rel(...)` already takes a
verb phrase as its label, and flowchart edges take one in `-->|"..."|`.
The one exception: a data-flow pipeline whose nodes are already
verb-phrase steps ("parse rows" → "write to DB") reads fine with bare
arrows; add a label only where a transition needs qualifying.

## The four grades

Use outlines/borders only — never change fill color — so grading doesn't
fight with any other meaning color might already carry (team ownership,
tech stack, etc.), and the diagram still reads in both light and dark
rendering. Each grade also gets a distinct stroke pattern, not just a
color, so the two easiest grades to confuse under red-green color
blindness (🔴 vs 🟢) are still distinguishable by pattern alone.

```
classDef ciGreen stroke:#2f9e44,stroke-width:3px;
classDef ciAmber stroke:#f08c00,stroke-width:3px,stroke-dasharray: 4 2;
classDef ciRed   stroke:#e03131,stroke-width:4px,stroke-dasharray: 2 2;
classDef ciGrey  stroke:#868e96,stroke-width:2px,stroke-dasharray: 1 3;
```

| Class | Meaning | Pattern |
|---|---|---|
| `ciGreen` | Matches the stated model | solid |
| `ciAmber` | Partially right / outdated / incomplete | medium dash |
| `ciRed` | Wrong / contradicted by the code | tight dash, thicker |
| `ciGrey` | Real, but absent from the stated model | fine dotted |

## Legend block

Always a diagram of its own, placed right after the reality diagram —
never a `subgraph` inside it. An embedded legend competes with the real
content for space and distorts the layout Mermaid computes for the actual
nodes. It renders the same whether the reality diagram is a flowchart or
C4, so this one flowchart block works in every report:

```mermaid
flowchart LR
    classDef ciGreen stroke:#2f9e44,stroke-width:3px;
    classDef ciAmber stroke:#f08c00,stroke-width:3px,stroke-dasharray: 4 2;
    classDef ciRed   stroke:#e03131,stroke-width:4px,stroke-dasharray: 2 2;
    classDef ciGrey  stroke:#868e96,stroke-width:2px,stroke-dasharray: 1 3;
    L1[Matches your model]:::ciGreen
    L2[Partially right / outdated]:::ciAmber
    L3[Wrong / contradicts your model]:::ciRed
    L4[Not in your model at all]:::ciGrey
```

## Architecture (C4-style)

Mermaid's C4 diagrams (`C4Context`, `C4Container`, `C4Component`) have a
built-in mechanism for exactly this — `UpdateElementStyle` and
`UpdateRelStyle`, originally meant for highlighting incremental changes.
Repurpose `$borderColor` for the grade:

```mermaid
C4Container
    title Reality: Order Processing System

    Person(customer, "Customer")
    Container(webApp, "Web App", "Next.js", "Customer-facing UI")
    Container(api, "API Service", "FastAPI", "Business logic + auth")
    ContainerDb(db, "Postgres", "Database", "Primary datastore")
    Container(worker, "Background Worker", "Python", "Async job processor")

    Rel(customer, webApp, "uses", "HTTPS")
    Rel(webApp, api, "calls", "HTTPS/JSON")
    Rel(api, db, "reads and writes", "SQL")
    Rel(api, worker, "enqueues jobs in", "Redis")

    UpdateElementStyle(webApp, $borderColor="#2f9e44")
    UpdateElementStyle(api, $borderColor="#f08c00")
    UpdateElementStyle(db, $borderColor="#2f9e44")
    UpdateElementStyle(worker, $borderColor="#868e96")
    UpdateRelStyle(api, worker, $lineColor="#868e96", $offsetY="-10")
```

Each `Rel` label reads as a clause: "Customer uses Web App", "API reads and
writes Postgres", "API enqueues jobs in Background Worker". Here `webApp`
and `db` matched the user's model (green), `api` was partially right
(amber — say why in the Reality column: e.g. "auth lives in a separate
service, not here"), and `worker` was a component the user never mentioned
at all (grey) — a gap, not a wrong belief.

Pick the C4 level to match scope: `C4Context` for a system landscape,
`C4Container` for one service's internals, `C4Component` for a single
module or feature. Don't reach for `C4Context` by default — it's the
least informative level and often too zoomed-out to show what actually
changed.

## Deployment

Mermaid has no dedicated "deployment diagram" type. A flowchart with
subgraphs as host/network boundaries reads clearly and renders everywhere
(GitHub, VS Code, most Markdown viewers). If your rendering target is known
to support newer Mermaid (`architecture-beta`), it's a fine alternative —
but flowchart is the safe default since `architecture-beta` support is
inconsistent across renderers as of this writing.

```mermaid
flowchart TB
    classDef ciGreen stroke:#2f9e44,stroke-width:3px;
    classDef ciAmber stroke:#f08c00,stroke-width:3px,stroke-dasharray: 4 2;
    classDef ciRed   stroke:#e03131,stroke-width:4px,stroke-dasharray: 2 2;
    classDef ciGrey  stroke:#868e96,stroke-width:2px,stroke-dasharray: 1 3;

    subgraph host["Docker host"]
        nginx["nginx :81"]:::ciGreen
        frontend["frontend :3000"]:::ciGreen
        server["server :8000"]:::ciAmber
        db[("postgres :5432")]:::ciGreen
    end

    subgraph ext["External network: shared_net"]
        graphics["graphics server"]:::ciGrey
    end

    nginx -->|"proxies page requests to"| frontend
    nginx -->|"proxies API calls to"| server
    server -->|"reads and writes"| db
    nginx -. "reaches over the shared docker network" .-> graphics
```

Group by physical/network boundary (host, container, external network,
managed cloud service), not by codebase folder — a deployment diagram
answers "what talks to what over the wire," which often cuts across the
source tree. The legend is a separate block after this diagram, not a
`subgraph` inside it.

## Grading a relationship, not a component

Some findings are about a *component* ("GraphicsServer exists and you never
mentioned it") and some are about *how two things connect* ("you said
polling, it's actually a push over Socket.IO"). The second kind belongs on
the edge, not on an extra node — inventing a node to carry an edge-shaped
finding (e.g. a `poll["polls every few seconds"]` node sitting on the arrow)
just moves the finding off the connection it's actually about and makes the
diagram harder to read as a graph. Grade edges directly instead.

**C4 diagrams** already have `UpdateRelStyle` for this (see the Architecture
section above) — use it on every `Rel(...)` whose claim was wrong or
outdated, and put the specific correction in the relationship's own label
text, phrased as a clause, not only in the Reality column:

```
Rel(server, webapp, "pushes live scores over Socket.IO, not polled")
UpdateRelStyle(server, webapp, $lineColor="#e03131", $offsetY="-20")
```

**Flowcharts** don't have a per-edge classDef, but Mermaid's `linkStyle`
targets an edge by its position in the diagram — edges are numbered `0, 1,
2, ...` in the order they're written in the source, across the whole
diagram (subgraphs don't restart the count):

```mermaid
flowchart LR
    api["Scoring API"]:::ciGreen
    display["Arena display"]:::ciRed

    api -->|"pushes new scores via Socket.IO, not polled"| display

    classDef ciGreen stroke:#2f9e44,stroke-width:3px;
    classDef ciRed   stroke:#e03131,stroke-width:4px,stroke-dasharray: 2 2;
    linkStyle 0 stroke:#e03131,stroke-width:3px
```

Reuse the same grade hex values as the node `classDef`s
(`#2f9e44`/`#f08c00`/`#e03131`/`#868e96`) so an edge's color means the same
thing a node's outline does. Before finalizing, recount the edges in
diagram-source order and double check the `linkStyle` index actually points
at the edge you mean — it's easy to miscount once a diagram has more than a
handful of arrows, and a `linkStyle` on the wrong index silently colors the
wrong connection instead of erroring.

Put the finding itself in the edge's label (`-->|"..."|`) so the diagram
states it directly; the table row's Reality column can then be a short echo
and its Evidence column is just the citation (file:line), not a restatement
of what the arrow already says.

## Data flow

A left-to-right flowchart following one or two concrete flows end-to-end
reads better than a diagram trying to show every possible path:

```mermaid
flowchart LR
    classDef ciGreen stroke:#2f9e44,stroke-width:3px;
    classDef ciAmber stroke:#f08c00,stroke-width:3px,stroke-dasharray: 4 2;
    classDef ciRed   stroke:#e03131,stroke-width:4px,stroke-dasharray: 2 2;
    classDef ciGrey  stroke:#868e96,stroke-width:2px,stroke-dasharray: 1 3;

    subgraph Entry
        upload["Accept CSV upload"]:::ciGreen
    end
    subgraph Transform
        parse["Parse + validate rows"]:::ciGreen
        persist["Write to DB"]:::ciGreen
    end
    subgraph Exit
        notify["Push update via WebSocket"]:::ciRed
        pdf["Generate PDF report"]:::ciGrey
    end

    upload --> parse --> persist --> notify
    persist --> pdf
```

The arrows are bare here because each node is already a verb-phrase step
(the pipeline exception from the top of this file) — add a label only where
a transition needs qualifying, e.g. `persist -->|"only confirmed rows"|
notify`. Here the user believed updates went out over raw WebSockets when
the code actually uses a different real-time mechanism (red — name what the
code uses instead in the Reality column), and PDF generation was a step
they didn't know existed at all (grey).

## "Your model" diagram

Use the same diagram type as the reality diagram (flowchart for
deployment/data-flow, C4 for architecture) but with **no classDef/styling
at all** — it's a faithful, neutral rendering of what the user said, not a
verdict. Don't grade this one; only the reality diagram carries grades.
