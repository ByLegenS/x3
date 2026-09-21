# Three modes of a work list

[The pages](INDEX.md) - [what x3 is](../README.md)

## Three modes of a work list

**What it catches:** a gate paying, on every run, for work that finished months
ago. A list has three modes, and the difference between them is not a label but
a **place**:

| Mode | Where it lives | When it is read |
|---|---|---|
| open | `file` | every run |
| deferred | `deferred` | only when the run asks (`-deferred`) |
| archived | `archive` | **never** |

```yaml
boxes:
  file: docs/OPEN-WORK.yaml
  deferred: docs/DEFERRED-WORK.yaml
  archive: docs/archive
```

Deferring is not a mark on a box. A box marked *"deferred"* inside the open list
is still read, still parsed, still counted — and reading it is the whole cost.
Measured on a real production Go application: one box step took **51.4 s**, and
of the 1 557 boxes it walked, **998 were already closed**. So a mode is a file,
and the file the run does not name is the file the run does not open.

The archive is the far end of that rule. It holds closed work in **monthly,
line-based parts** (`archive/2026-09.jsonl`) rather than one growing YAML
document, because a single document is rewritten whole every time it grows, while
last month's part is never touched again. The gate does not measure it, does not
count it, and cannot go red because of it — its only purpose is that somebody can
search it later.

Both fields belong to the machine-written list and are refused beside `sources`:
deferring work means not reading it, and a document is read either way. A path
that is absolute or climbs out of the tree is refused, and so is a `deferred`
that names the same file as `file` — one box would be read twice, and the two
readings could never disagree out loud.

A run says which modes it read, in the report and in the line it prints:

```
x3 boxes: 2 box(es) in work.yaml + later.yaml - 2 open, 0 done, 0 silent, 0 moved - 0 block, 0 warn
x3 boxes: 1 of them deferred, from later.yaml
```

A run that did not ask says nothing about deferred work — a count that is missing
part of the list cannot announce that by being missing.

**Moving a box between the three modes.** A mode is a place, so moving work between modes means moving the **box**, and
the engine moves it:

```
x3 boxes -close <ids> | -close-from <file> | -close-finished
x3 boxes -defer <ids> | -defer-from <file>
x3 boxes -resume <ids> | -resume-from <file>
```

Each takes comma-separated ids, or one id per line from a file, the same way
`-holds` and `-holds-from` do. One move runs at a time: a run that closed and
deferred at once could not say which list a box left.

**Closing measures, it does not record.** `-close` reads the box's `done:`
criteria and measures them *now*. A box whose criteria do not all hold is
refused by name, and the criterion that did not hold is printed — writing
`state: done` by hand is what this refuses to be:

```
x3 boxes -close a-box-a-command-can-open
BLOCK a-box-a-command-can-open: box_unproven
	the box was asked to close and its criteria do not hold; done without proof is not done
	unmet: pattern "add := fs\.String\("add"" in cmd/x3/main.go
	unmet: pattern "A box a command opens" in docs/CAPABILITIES.md
x3 boxes -close: 1 asked - 0 moved to x3/archive, 1 refused, 0 unknown - 136 box(es) left in docs/OPEN-WORK.yaml
```

Only the boxes asked about are measured: closing two boxes does not run every
test the list names. An id no list holds is `box_unknown` and is red as well —
a misspelled id is a box somebody believes was moved.

**Closing in bulk is a measurement, not a name pattern.** `-close-finished`
closes every open box the run reports as *finished but still open*, so the
boxes it picks are the ones the gate already names, and each one is measured
again on its way out. A pattern over ids would one day close a box nobody
measured:

```
x3 boxes -close-finished
x3 boxes -close: a-work-list-whose-modes-can-be-moved-between -> x3/archive/2026-09.jsonl
x3 boxes -close: 1 asked - 1 moved to x3/archive/2026-09.jsonl, 0 refused, 0 unknown - 136 box(es) left in docs/OPEN-WORK.yaml
```

**Deferring and resuming measure nothing**, because they change how often work
is read, not whether it is done:

```
x3 boxes -defer words-a-comment-may-not-carry-either
x3 boxes -defer: 1 asked - 1 moved to docs/DEFERRED-WORK.yaml, 0 refused, 0 unknown - 133 box(es) left in docs/OPEN-WORK.yaml
x3 boxes -resume words-a-comment-may-not-carry-either
x3 boxes -resume: 1 asked - 1 moved to docs/OPEN-WORK.yaml, 0 refused, 0 unknown - 0 box(es) left in docs/DEFERRED-WORK.yaml
```

A move rewrites neither list: the box's own YAML node is lifted out of one file
and put into the other, so the comments, key order and formatting of every box
that did not move stay byte for byte. Measured on this repository: deferring one
box out of 134 changed 12 lines, all of them that box. A list re-serialized on
every move would hide a one-box change inside a hundred-line diff.

**Searching the archive.** The archive is written by `-close` and read by nothing else — not by the gate,
not by a count, not by a baseline. One command reads it back:

```
x3 boxes -search <pattern>
```

The pattern is a regular expression and it is matched against what a person
would search for: the box's id, its title, and the criteria that were met when
it closed. It is not matched against the raw line — a search for `pattern`
would otherwise find every record, because that word is a field name.

```
x3 boxes -search "three modes"
a-work-list-whose-modes-can-be-moved-between	2026-09-21T05:54:44Z	x3/archive/2026-09.jsonl
	commands that move a box between the three modes - closing measures the box, deferring only changes how often it is read, and the archive can be searched
x3 boxes -search: 1 record(s) match "three modes" - 1 record(s) read from 1 part(s) of x3/archive
```

The count of records **read** is printed beside the count matched: a search that
found nothing and a search that opened nothing are not the same answer, and an
archive directory that does not exist yet reads zero parts rather than failing.

Each record carries the box's criteria as they were written, so a box taken out
of the archive can go back into a list with what proved it:

```json
{"id":"gate-hook-flags","closed":"2026-09-21T05:54:44Z","from":"docs/OPEN-WORK.yaml",
 "met":["pattern \"no git, no silence\" in x3.yaml"],
 "done":[{"when":"pattern","match":"no git, no silence","sources":["x3.yaml"]}]}
```

<!-- x3-dist version=v0.260.0 capabilities=8004fa03548574d60563f63f663d52673559fe0003df4bd6e444d702ee9ac59d template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
