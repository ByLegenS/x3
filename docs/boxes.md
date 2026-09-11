# Open work, measured

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes`

**What it catches:** a work list drifting — work that finished while the box
stayed open, and a box closed by somebody who only meant to finish it. Each box
carries the criteria that would prove it done, and the gate asks both questions.

```
x3 boxes [-config <file>] [-out <file>] [dir]
```

```json
{ "boxes": [
    { "id": "arch-containment", "title": "the containment rule kind", "state": "open",
      "done": [ { "when": "file", "path": "internal/arch/containment.go" },
                { "when": "pattern", "sources": ["docs/CAPABILITIES.md"],
                  "match": "### `containment`" } ] } ] }
```

The list lives in its own file (`{ "boxes": { "file": "docs/OPEN-WORK.json" } }`)
because it changes weekly while the configuration changes yearly. That is this
repository's own list, and `check.ps1` runs this command against it.

### A box that is not needed yet

Some work is owed only once something else happens — a second tenant, a version
bump that has not landed. `when` says what makes it due:

```json
{ "id": "second-voice-application", "state": "open",
  "when": [{ "when": "pattern", "sources": ["apps/*/kind.json"], "match": "\"voice\"" }],
  "done": [{ "when": "file", "path": "docs/migration-b.md" }] }
```

While the condition does not hold the box waits: open without being red, and
counted. The rule that does **not** relax is the other one — **a box closes
because the work was done, never because it stopped being needed.**

### Both directions, or neither

| State | Criteria | Result |
|---|---|---|
| `open` | all met | **red** — `box_finished`: the work is done, the list is stale |
| `done` | any unmet | **red** — `box_unproven`, each unmet criterion named (`box_unmeasured` when none of them could be measured) |
| `open` | some unmet | green |
| `done` | all met | green |

Asking only the second lets a list fill with finished work; asking only the first
leaves closing without proof free.

### The criteria

See **boxes criteria fields** in [REFERENCE.md](../REFERENCE.md#boxes-criteria-fields).

`match` is read with `^` and `$` bound to a **line**
([how](patterns.md#how-a-pattern-is-read)). The sharp edge is `absent`: a `pattern` that
stops matching goes loudly red, but an `absent` that stops matching goes **green
without measuring anything**. It is also what makes deletion provable — "the old
call site is gone" is exactly the sentence that becomes true when that work
finishes.

`equals` is required on `sql`, because a query that only has to *run* is answered
by an empty table; the DSN is read from the environment by name and never written
in the list, and a `driver` this binary never registered is a configuration error
(exit `2`) rather than a criterion that quietly could not be measured. On
`manual`, `by` and `seen` are required because a criterion without them is an
intention, and every run prints how many criteria are manual — a ratio that grows
is a list drifting back to nobody checking. Fields belong to exactly one
criterion, and a foreign one is refused rather than ignored in silence.

#### The exit code is half a command criterion

A runner asked for a test that was never written exits `0`; a suite whose every
case skipped itself prints `PASS` and exits `0`. Both read as "finished", which
means **writing the criterion is enough to close the box**.

```json
{ "when": "command", "command": "go", "args": ["test", "-v", "./..."],
  "output": { "must": ["--- PASS"], "mustNot": ["no tests to run"],
              "retry": { "when": ["no tests to run"], "args": ["-tags", "integration"] } } }
```

`must` and `mustNot` are plain substrings searched in **both streams together** —
a runner's "I found nothing to run" is usually on stderr. `retry` runs the
criterion **once** more, only when the first output says one of `when`, and the
second run *replaces* the first: "either attempt may hold" would be an escape
hatch. Nothing here is specific to any runner.

#### A criterion that cannot run, and one that ran and measured nothing

A `sql` criterion whose DSN is empty, or a `command` that cannot start, is
**unmet** — never met, with the reason beside it. So is a run that *did* start and
measured nothing: a suite whose cases all skipped themselves exits `0` and prints
no proof. Which sentence means "I skipped" is the project's to declare.

```json
"output": { "must": ["--- PASS"], "mustNot": ["no tests to run"], "unmeasured": ["--- SKIP"] }
```

`unmeasured` is read **only** where the expectation did not hold and `mustNot` said
nothing: a rejected output *was* measured, so a stale criterion cannot file itself
as unmeasurable. One text written in both `must` and `unmeasured` is a
configuration error (exit `2`).

**Nothing turns green.** A closed box whose every unmet criterion could not be
measured is red under its own code, `box_unmeasured`; a single unmet criterion that
*was* measured keeps the box at `box_unproven`, because a measured red may not hide
behind an unmeasured one. Both counts stand on the human line and in
`summary.unmeasured` / `summary.unmeasuredBoxes` — one field for "how many boxes did
this run fail to measure" — and neither code may be frozen into a baseline. A
`when` condition that could not be measured does **not** postpone a box: an
unmeasurable condition would silence both directions at once.

A list with no boxes is `empty_scope`: a list that says nothing does not say
everything is finished.

#### The shell a run was measured in

Two runs of the same tree, at the same version, can give two different numbers,
and the reason is almost never the tree: a check that reads an environment
variable skips itself when the shell does not carry it, the output never says
"passed", and the box is counted unproven. Read side by side, that difference
looks like a regression. So the report says where it was measured.

```json
{ "boxes": { "environment": ["APP_TEST_DSN", "APP_LIVE_TOKEN"] } }
```

`summary.environment` names each variable and whether it was **set** — never its
value, because a report that carried one would be the leak it exists to prevent.
The list is the declared names plus every variable a criterion already names for
itself (`dsnEnv`), so a connection string is not declared twice. Which variable a
*command* reads is the project's to say: the engine cannot open a test and guess.
Nothing here changes a verdict — what a shell ought to carry is not the engine's
judgement — and a run with no variable to speak of prints no line at all.

`summary.blind` splits the unmeasured count by cause: `environment` (a variable
the criterion needs is empty), `connection` (the query did not run), `command`
(the command never started), `output` (the run said it did not measure). One
total cannot tell a closed database from a skipped suite, and the difference
between two runs is exactly what a total hides. The causes sit in `summary`, not
in a finding, so a baseline still reads the same bytes from the same tree.

### Asking many criteria in one process

A list's cost grows with the number of **processes**, not of criteria.

```json
"runs": { "when": "command", "prefix": ["go", "test", "-v"], "output": { "must": ["--- PASS"] },
          "batch": { "select": "-run", "join": "|", "mark": "^\\s*(?:=== \\w+|--- \\w+:)\\s+(\\S+)" } }
```

Criteria differing only in `select`'s value are joined with `join` into one call.
`mark`'s first capture group is the name a line belongs to, matched against a
selector as an unanchored expression; **a line matching no name is shared**, so a
compile error blinds the whole call, not one member of it.
**Batching can only ever produce green:** a member is met when its *own* lines
meet `output` in a call that exited `0`; every other verdict comes from the
criterion's own process, the path a project declaring no batch already takes. A
non-zero exit settles nothing, so the set narrows to the members that held and is
asked again — one failing test costs one extra call, not a process each. `batch`
needs `output.must`: nothing asked for is met by no lines at all, the hole
`output` exists to close. Criteria in one call see each other.

`{ "boxes": { "workers": 8 } }` runs those calls in parallel. On a real list —
2297 boxes, 1307 criteria, 185 starting a process over 52 packages, sixteen
cores — nothing declared is 129.6 s / 235 processes, `batch` 111.8 s / 205,
`workers: 8` 54.1 s / 235, both **47.9 s / 205**, and all four reports are
byte-identical. The honest row is the second: 30 processes fewer and no reliable
time, because 71 of those criteria do not hold. **Parallelism buys the waiting;
batching buys a process, only for the criteria that hold**; `sql` stays serial.

### A list written as a document

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

#### The engine does not know what "in progress" means

A mark gets a `name`, which the report uses, and a `means`, which is the only
thing the engine acts on: `open` (all criteria met is red), `done` (any unmet is
red), `silent` (neither direction is asked). So whether "waiting on somebody"
goes red when its proof already stands is one line of the project's
configuration. `silent` is an escape hatch, and a `silent` state no box carries
is `dead_state` and red.

#### The record a state must carry

An in-between state is a claim, not a condition. `requires` names the fields that
must sit in the item's body, in the project's own words; a missing one is
`box_record`, and so is a field so short it is a way of not answering —
`minLength` sets the floor, and `left: -` does not clear it. A field line may
carry any leading decoration; what counts is a name, a colon and something after
it. The body of an item is everything indented under it, or — for an item written
as a heading — everything to the next heading. Checkboxes inside a fenced code
block are examples, not work.

#### Writing a criterion in prose

`criterion.key` opens a criterion line and `kinds` maps the project's word onto
one of the six criteria. Everything a criterion needs but a document should not
repeat — the DSN variable, the runner `prefix`, the separators — lives in the
kind, not in the line.

See **boxes criteria written in prose** in [REFERENCE.md](../REFERENCE.md#boxes-criteria-written-in-prose).

Place and expression split at the first space, so a path containing one is
quoted, and **a quote that never closes is a configuration error** on that line
rather than a path quietly split in two:

```
criterion: contains "docs/design notes/READER.md" the parser is here
```

#### What the reader refuses

`box_unknown_state` is a mark the configuration never declared. `box_unlisted` is
`* [ ]`, `+ [ ]` or `1. [ ]` — drawn like a checkbox, collected by nothing, which
is the quiet one: the work was written down and is in no list, so nobody will
come looking for it.

#### Regions that are not work

A note that keeps a work list usually also shows **how an item is written**, drawn
with the same checkboxes. Fenced code blocks are skipped because that is
Markdown's own writing; every other marker is declared:

```json
"markdown": { "examples": [ { "open": "^<!-- EXAMPLE -->", "close": "^<!-- /EXAMPLE -->" } ] }
```

Both are required, and the marker lines are skipped with everything between them.
A region that opens and never closes is `example_unclosed`; one that opens in no
document at all is `dead_example`.

### A list that is adopted gradually

A gate that reds a thousand times on its first run is switched off on its second,
so `boxes` reads a baseline exactly like `comments` or `secrets`:

```
x3 boxes -baseline baselines/boxes.json -update-baseline   # freeze what stands now
```

What may be frozen is **how the list is written today** — `box_uncovered`,
`box_record`, `box_unlisted`, `box_unknown_state`, `box_owner`, `box_moved`,
`box_suspect`. What may **never** be frozen is what the list *claims*:
`box_finished`, `box_unproven`, `box_unmeasured` and `box_dead_selector`
(freezing them makes finished work sit open forever, closing without proof free,
and a gate that measures nothing green), `empty_scope`, and the gate's own health codes. A baseline buys time to write the criteria; it does not buy permission to
stop asking the two questions.

### A list that is finished, and where it goes next

Every criterion so far reads one **item**. This one reads a whole **document**,
because a list can fail as a list while every item in it is written correctly.

```json
{ "markdown": { "retire": { "from": "ongoing", "to": "done",
    "bare": true, "empty": true,
    "few": [ { "open": 2, "finished": 1 }, { "open": 5, "percent": 75 } ] } } }
```

`from` is the part of a path that says "this document is open work", `to` is what
replaces it, and **both are the project's words**. `bare` finds a document among
the open lists carrying no box at all (`list_has_no_box`), `empty` one where
nothing is open any more (`list_finished`), and `few` one where a threshold holds
(`list_nearly_finished`).

`bare` is why the rule walks **documents** rather than boxes: a file with no box
produces nothing to walk past, so the one failure that leaves work completely
invisible is exactly the one a box-by-box gate cannot see — and its finding does
not say "move this", because what it needs is boxes. Each `few` threshold is "at
most this many open, and this much finished", and **the finished half is
required**: `open` alone would retire a one-item plan nobody has started, filing
work nobody has begun under work that is done.

**The engine does not write the list out.** Turning the report into a page
somebody reads over breakfast is formatting, and a gate that also published
documents would own two contracts.

### Work that moved rather than finished

An item is sometimes closed because it was written down somewhere else: the work
did not finish, its **place** changed.

```json
{ "markdown": { "moved": { "match": "^\\s*moved to `([^`]+)`\\s*$", "roots": ["docs"] } } }
```

An item whose body carries that line is outside **both** directions — its criteria
are not run, and it is not `box_uncovered` either. The pattern is the project's
own and its **first capture group** is where the work went. It is not free: saying
"moved" would otherwise be the cheapest way to silence a criterion, so the target
has to **exist**, and one nothing can be found at is `box_moved`. A marker that
matches nothing is **not** red — the item simply keeps its criteria, so the
failure is loud rather than blind.

### A scope that can be narrowed

```json
{ "boxes": { "sources": ["docs/**/*.md"], "exclude": ["docs/external/**"] } }
```

The law is `arch`'s and `freeze`'s, shared in one place: a pattern that takes no
document out is `dead_exclusion`, an empty list is refused, and a scope holding no
box is `empty_scope`. `exclude` narrows documents, so it may not be written beside
`file` — a machine-written list has no scope to narrow.

### A document that can carry a signature

A `manual` criterion holds when `signed` records that the looking happened.

```markdown
- [x] the installer works on a clean machine
      criterion: by-hand the release owner - the installer runs on a clean machine - signed 2026-09-07
```

`separator` splits who looks from what they see; `signed` splits what they see
from the record that they did. Both are the project's own words, and if the kind
declares no `signed` marker a manual criterion can never hold — correct rather
than convenient.

### Who a manual criterion may wait on

```json
{ "boxes": { "manual": { "denyBy": ["the person who owns this project"] } } }
```

A `manual` criterion whose `by` matches one of those names is `box_owner`. This is
a **prohibition**, not an escape hatch, so it does not shout when it matches
nothing — a rule that catches nothing is good news.

<!-- x3-dist version=v0.96.0 capabilities=890b3ee4cf9878440f2cd2d8516be453e9d75846ffce0092312b5bb1236761b4 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
