# A work list written as a document

[The pages](INDEX.md) - [what x3 is](../README.md)

## A list written as a document

Most projects keep their open work in their documents, as Markdown checkboxes
with more than two states. `sources` reads the list that way — and `file` and
`sources` cannot both be written.

```json
{ "boxes": { "sources": ["docs/**/*.md"], "markdown": {
    "states": [
      { "mark": " ", "name": "open",  "means": "open" },
      { "mark": "~", "name": "doing", "means": "open", "requires": ["done-so-far", "left"] },
      { "mark": "x", "name": "done",  "means": "done" } ],
    "criterion": { "key": "criterion", "kinds": {
      "exists":   { "when": "file" },
      "contains": { "when": "pattern" },
      "passes":   { "when": "command", "prefix": ["go", "test", "-v"],
                    "output": { "must": ["--- PASS"] } },
      "by-hand":  { "when": "manual", "separator": " - ", "signed": " - signed " } } },
    "minLength": 4 } } }
```

```markdown
- [~] the report that names a state
      done-so-far: the reader is written
      left: the report still prints only the mark
      criterion: exists docs/reader.md
```

### The engine does not know what "in progress" means

A mark gets a `name`, which the report uses, and a `means`, which is the only
thing the engine acts on: `open` (all criteria met is red), `done` (any unmet is
red), `silent` (neither direction is asked). So whether "waiting on somebody"
goes red when its proof already stands is one line of the project's
configuration. `silent` is an escape hatch, and a `silent` state no box carries
is `dead_state` and red.

### The record a state must carry

An in-between state is a claim, not a condition. `requires` names the fields that
must sit in the item's body, in the project's own words; a missing one is
`box_record`, and so is a field so short it is a way of not answering —
`minLength` sets the floor, and `left: -` does not clear it. A field line may
carry any leading decoration; what counts is a name, a colon and something after
it. The body of an item is everything indented under it, or — for an item written
as a heading — everything to the next heading. Checkboxes inside a fenced code
block are examples, not work.

### Writing a criterion in prose

`criterion.key` opens a criterion line and `kinds` maps the project's word onto
one of the six criteria. Everything a criterion needs but a document should not
repeat — the DSN variable, the runner `prefix`, the separators — lives in the
kind, not in the line.

See **boxes criteria written in prose** in the [boxes reference](boxes-reference.md#boxes-criteria-written-in-prose).

Place and expression split at the first space, so a path containing one is
quoted, and **a quote that never closes is a configuration error** on that line
rather than a path quietly split in two:

```
criterion: contains "docs/design notes/READER.md" the parser is here
```

### A criterion written inside a table

A criterion line is recognised from the **start of the line**, past indentation,
quoting, arrows and bold — and a table pipe is none of those. A project that
keeps its criteria in a table would therefore never be measured at all: the box
looks uncovered, its name lives in no criterion, and the day that name is removed
the measurement dies without a word.

So every body line that is a Markdown table row is **also** read cell by cell:

```
| criterion: contains parser.go the parser is here | why it matters |
```

Only a cell whose own first word is the criterion key is read. A criterion
**quoted** inside a sentence — backticked, or with any other word before it — is
prose, and prose is not a criterion; the engine does not invent one from a
document that merely talks about another document's criteria.

An **escaped pipe** (`\|`) does not separate a cell and reaches the criterion as
the pipe itself, because a pattern's alternation can only be written in a table
that way, and reading the escaped form would measure something the project never
wrote. A line carrying a single pipe is prose, not a row.

### What the reader refuses

`box_unknown_state` is a mark the configuration never declared. `box_unlisted` is
`* [ ]`, `+ [ ]` or `1. [ ]` — drawn like a checkbox, collected by nothing, which
is the quiet one: the work was written down and is in no list, so nobody will
come looking for it.

### Regions that are not work

A note that keeps a work list usually also shows **how an item is written**, drawn
with the same checkboxes. Fenced code blocks are skipped because that is
Markdown's own writing; every other marker is declared:

```json
"markdown": { "examples": [ { "open": "^<!-- EXAMPLE -->", "close": "^<!-- /EXAMPLE -->" } ] }
```

Both are required, and the marker lines are skipped with everything between them.
A region that opens and never closes is `example_unclosed`; one that opens in no
document at all is `dead_example`.

<!-- x3-dist version=v0.153.0 capabilities=623ccd05726c1539c169bc853f4869d52533d5c7fa7810be91e81b097e7bfbb6 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
