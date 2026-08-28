# Writing style for the report

Five rules from Strunk's *The Elements of Style*, picked for the kind of
prose this skill produces: the TL;DR bullets, the diagram captions, the
one-line restatement of the model, and the Reality column of the findings
table. Apply them there. The diagrams carry the load; the words around them
should be short, plain, and specific.

### Omit needless words

Every word should tell. Cut "the fact that", "in order to", "it is
important to note that", "very", "really", "quite", "interesting". Combine
sentences that step through one idea in fragments.

- *There is no polling loop present in the live path at all* → *The live path never polls.*

### Use definite, specific, concrete language

Name the file, the port, the function, the protocol. Never "this seems
off", "something's different here", "the config looks wrong".

- *nginx is on a different port than expected* → *nginx listens on 81, not 80.*

### Put statements in positive form

Say what is, not what isn't. Reserve "not" for genuine denial or
contrast, not for hedging.

- *the display doesn't poll* → *the display holds an open Socket.IO connection.*
- *the volume isn't ephemeral* → *the volume persists between events.*

### Use the active voice

A named actor doing something reads faster than a passive construction,
and it forces you to identify the actor.

- *PDFs are generated server-side* → *the server renders PDFs with fpdf2.*
- *the schema is validated on upload* → *`parse_rows()` validates the schema on upload.*

### Express co-ordinate ideas in similar form

Keep the TL;DR bullets grammatically parallel with each other, and keep
each column of the findings table in a consistent shape down its rows.
Parallel form lets the reader compare rows at a glance instead of
re-parsing each one.
