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

### What "climbs out of the tree" covers

The refusal is written against **every** way of leaving the tree, not against the
spelling `../`. Measured on 2026-09-21, the four forms the first version let
through:

| Written as | Why it slipped | Now |
|---|---|---|
| `..` | bare, so it carries no `../` prefix | refused |
| `a/../..` | cleans down to `..` — same hole, other spelling | refused |
| `..\yan` | `\` is a separator **only on Windows**, so one settings file climbed on one operating system and named a file on the other | refused on both |
| `C:/elsewhere`, `c:here` | a drive letter is not "absolute" on Linux, and a drive-relative `c:here` is not absolute on Windows either | refused |

```yaml
boxes:
  file: x3/work/open-work.yaml
  archive: ..            # refused: climbs out of the tree
```

```
x3 boxes: x3.yaml: boxes: archive ".." climbs out of the tree; the work of a
repository is kept by that repository
```

`.` is **not** a climb and is not refused: it names the tree itself, not
somewhere outside it. What the rule protects is that a work list belongs to the
repository carrying it — a path reaching past the root moves that list onto the
machine, where no other checkout can read it.

### One list, or one per region

`file` and `deferred` may also be **globs**, and then every file the pattern
matches is read and the boxes merge into one list:

```yaml
boxes:
  file: '**/x3/work/open-work.yaml'
  deferred: '**/x3/work/deferred-work.yaml'
  archive: x3/work/archive
```

```
x3 boxes: 5 box(es) in **/x3/work/open-work.yaml (3 file(s)) - 5 open, 0 done, 0 silent, 0 moved - 0 block, 0 warn
```

The component a box belongs to is then read from **where it stands** — a box in
`apps/beta/x3/work/open-work.yaml` is that application's — and no field says so.
A field drifts from its file the day somebody moves the box; a place cannot. A
gate step that reads one component reads one file, so it stops paying for work
that belongs to everybody else.

Three rules keep the split honest:

- A box is moved **beside itself**: `-defer` writes into the deferred list of
  the same folder, never into the first one the engine happened to read. The two
  patterns may differ in their **file name** only — one that differs in a folder
  above it is refused, because the engine would be guessing which place answers
  which. A plain path on either side keeps its plain meaning: many lists may
  defer into one.
- **An identity is unique across lists, not inside one.** The same id in two
  files is refused, and the refusal names both: one identity in two lists is two
  boxes read as one.
- **The archive is not split.** No run reads it, so splitting it buys nothing and
  costs the only thing it is for — finding closed work again, in one place.

A plain path stays a plain path, and a project with one list walks no tree.

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
x3 boxes -close: a-work-list-whose-modes-can-be-moved-between: docs/OPEN-WORK.yaml -> x3/archive/2026-09.jsonl
x3 boxes -close: 1 asked - 1 moved to x3/archive/2026-09.jsonl, 0 refused, 0 unknown - 136 box(es) left in docs/OPEN-WORK.yaml
```

**Deferring and resuming measure nothing**, because they change how often work
is read, not whether it is done:

```
x3 boxes -defer words-a-comment-may-not-carry-either
x3 boxes -defer: words-a-comment-may-not-carry-either: docs/OPEN-WORK.yaml -> docs/DEFERRED-WORK.yaml
x3 boxes -defer: 1 asked - 1 moved to docs/DEFERRED-WORK.yaml, 0 refused, 0 unknown - 133 box(es) left in docs/OPEN-WORK.yaml
x3 boxes -resume words-a-comment-may-not-carry-either
x3 boxes -resume: 1 asked - 1 moved to docs/OPEN-WORK.yaml, 0 refused, 0 unknown - 0 box(es) left in docs/DEFERRED-WORK.yaml
```

A move rewrites neither list: the box's own YAML node is lifted out of one file
and put into the other, so the comments, key order and formatting of every box
that did not move stay byte for byte. Measured on this repository: deferring one
box out of 134 changed 12 lines, all of them that box. A list re-serialized on
every move would hide a one-box change inside a hundred-line diff.

**An identity is minted, not derived.** A box identity carries no place: no path,
no colon, no line. The loader refuses one that does, because a derived identity
turns into a *different* identity the moment its source moves — the document is
renamed, the heading is corrected — and every record keyed to it (the archive,
a baseline, someone's note) silently points at nothing. An identity that carries
a path also invites the reader to derive the box's category from it, and that
inference starts answering wrongly the day the work moves.

So the engine mints one instead: twelve random hex digits, written into the list
once and never recomputed. Each mint is checked against the identities already
recorded, so a collision inside a list is not unlikely but impossible; the width
only keeps the retry from mattering (unchecked, the birthday chance over 516
boxes would be 5e-10, and 2e-7 over ten thousand).

```
x3 boxes -renumber
```

replaces every placed identity with a minted one and writes the old name beside
it as `was:`. The mapping is built **once** and every carrier reads it — the open
list, the deferred list and each archive part — so a box named in two of them
comes out of the migration with one identity, not two:

```
x3 boxes -renumber
69ac3e18d797	docs/PLAN.md:67f028ef1d9d	work.yaml
7d9b16680352	docs/PLAN.md:d162463f2a72	later.yaml
x3 boxes -renumber: 2 id(s) minted for 2 old name(s), 0 already minted - 3 file(s) written: work.yaml, later.yaml, archive/2026-09.jsonl
```

Two names, three files. Nothing is regenerated: only the `id` scalar changes and
a `was` key is inserted after it, so body, criteria, order and comments stay byte
for byte, and an archive record keeps every field it was written with.

An archive record is **read the way it is parsed**, not matched as text. The
migration asks the same JSON reader that loads the part where the record's own
`id` ends, so `{"id":"b-1",…}` and `{"id": "b-1", …}` are one record, and the
bytes after the identity are carried over untouched. A file with two reading
paths — a parser in one place, a text match in the other — is a file the engine
cannot promise to read back: the match breaks on whitespace the parser never
noticed. What the migration still refuses is a record that does not open with an
identity at all:

```
x3 boxes -renumber: archive/2026-09.jsonl:1: the record does not open with its own id; the archive is written by the engine and read back the same way
```

`was:` is a headstone, and the engine derives nothing from it — not a category,
not a place, not an order. It exists so that the old name still answers. A move
asked by the name the box carried before the migration finds it, and the run
names it by the identity it carries now:

```
x3 boxes -defer docs/PLAN.md:67f028ef1d9d
x3 boxes -defer: 69ac3e18d797: work.yaml -> later.yaml
```

Asked a second time the migration mints nothing — a minted identity is never
reminted, so the command is safe to run on a list that is already clean:

```
x3 boxes -renumber: 0 id(s) minted for 0 old name(s), 2 already minted - 0 file(s) written
```

A hand-written readable id (`a-package-that-embeds-is-never-remembered`) is a
perfectly good identity and the loader leaves it alone; what it refuses is a
*place* standing in for a name.

**Searching the archive.** The archive is written by `-close` and read by nothing else — not by the gate,
not by a count, not by a baseline. One command reads it back:

```
x3 boxes -search <pattern>
```

The pattern is a regular expression and it is matched against what a person
would search for: the box's id, the name it **was** known by before a migration,
its title, its **body**, and the criteria that were met when it closed. It is not matched against the raw line — a search for
`pattern` would otherwise find every record, because that word is a field name.

The body is the half that makes the archive answerable. A word a person
remembers is almost never in the title:

```
x3 boxes -search "silenced digit"
rounded-price	done	2026-09-21T07:35:44Z	archive/2026-09.jsonl
	the gate must refuse a price nobody can read
x3 boxes -search: 1 record(s) match "silenced digit" - 1 record(s) read from 1 part(s) of archive
```

The same archive line read by an engine whose records carry no body answers `0
record(s) match` — the record is there, the sentence is not.

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

<!-- x3-dist version=v0.279.0 capabilities=2d4dbaa4fd0947e09be46fbdc475aa6d2c15d07d65a13b06f643f87f8c988536 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
