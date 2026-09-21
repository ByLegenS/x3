# The history a document carries

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -keep`

**What it catches:** a migration that can only carry the work it can measure,
and silently loses the rest when the source documents are deleted.

```
x3 boxes -keep [-config <file>] [-out <file>] [dir]
```

`-export` asks *"which of these boxes can a run measure?"* and writes those.
That is the right question for a list the gate reads every run, and it is the
wrong question for history. Measured on a production tree: of 1 557 box lines
written inside 201 finished documents, **913 (59%) reached no export at all** —
they carried no criterion, so no machine-written list could ever hold them. Had
the documents been deleted after the export, those 913 records would be gone.

`-keep` asks nothing. It reads the same documents and writes **every** box into
the archive as it stands.

```
x3 boxes -keep
x3 boxes -keep: 4 box(es) read from 1 document(s) in docs/*.md - 4 written to archive/2026-09.jsonl, 0 already there
x3 boxes -keep: 1 open, 3 done - 2 with no criterion, 0 moved away; nothing was measured and nothing was closed
```

### It records, `-close` measures

The two are opposites, and both are right.

`-close` measures the box's `done:` criteria *at that moment* and refuses a box
whose criteria do not all hold, because writing "finished" beside open work
stands in for finishing it. Applied to history, that rule asks the wrong
question: a criterion may have changed since the work was done, and a criterion
that no longer holds does not mean the work was never done. Measured on the same
production tree, `-close` over 54 historical closed boxes moved 45, **refused 8**
and could not measure 1 — a 15% refusal rate, none of it unfinished work.

So `-keep` measures nothing. A criterion that names a file deleted years ago is
written into the archive beside its box, exactly as the document has it:

```
x3 boxes                       # the measuring path, same documents
BLOCK a840843a5aca: box_unproven
BLOCK 79c8e978eeac: box_uncovered

x3 boxes -keep                 # the recording path, same documents
x3 boxes -keep: 4 box(es) read from 1 document(s) in docs/*.md - 4 written to archive/2026-09.jsonl, 0 already there
```

### A box keeps its state

A document called *finished* still holds open boxes — 139 of them in the pilot
tree. Writing them down as closed erases the work; moving them into the list the
gate reads every run undoes the whole point of not reading finished work. The
record says which one it is, and the search finds both:

```
x3 boxes -search "pilot tree"
475a7b66f124	done	archive/2026-09.jsonl
	no criterion was ever written beside this one
1e6ef0bc569d	open	archive/2026-09.jsonl
	this box is still open inside a document called finished
x3 boxes -search: 2 record(s) match "pilot tree" - 4 record(s) read from 1 part(s) of archive
```

A historical record carries no closing time, and that blank is deliberate: the
engine does not know when past work finished, and writing today's date there
would invent a measurement. The month the record entered the archive is the name
of the part file it is in.

### Running it twice writes nothing twice

A box read out of a document has no identity until it is read: the engine
**mints** one, so the same box gets a different id on every reading (measured:
`a0caa1c8b9f7` on one run, `9ec8163fdfea` on the next). An archive that
recognised records by id would therefore write the whole tree again on a second
run.

Each record carries a **signature** instead — the document it came from, its
title and its body, hashed. Line numbers are deliberately not part of it: a
sentence added at the top of a document would shift every signature below it and
the next run would write everything a second time.

```
x3 boxes -keep
x3 boxes -keep: 4 box(es) read from 1 document(s) in docs/*.md - 4 written to archive/2026-09.jsonl, 0 already there
x3 boxes -keep
x3 boxes -keep: 4 box(es) read from 1 document(s) in docs/*.md - 0 written to archive/2026-09.jsonl, 4 already there
```

*Written* and *already there* are printed separately on purpose: the answer to a
second run is not "I wrote nothing" but "all of it was already written", and one
number for both would make an empty tree and a complete archive print the same
line. A migration of this size can therefore be stopped halfway and restarted —
which is not a convenience but a condition, because a partial archive that could
not be resumed would be unrecoverable mess.

### A line no collector can read is red

A line drawn like a box that no collector reads cannot be archived, and it goes
with the document when the document is deleted. That is the one mistake this
migration cannot take back, so the run says it by name and exits `1`:

```
BLOCK docs/PLAN.md:6c1f3a: box_unlisted
	this line is drawn like a box and no collector reads it
	at: docs/PLAN.md:12
x3 boxes -keep: 1 line(s) are drawn like a box and no collector reads them; they are not in the archive and they go with the document - read them above before deleting a source
```

### Settings

`-keep` reads `sources:` (the documents) and writes to `archive:`. Both may be
written together: the archive is not a *mode* of a list — it is a place outside
every list, and a list kept inside documents has a history too. Only `deferred:`
belongs to a machine-written list, because deferring work means not reading it
and a document is read either way.

<!-- x3-dist version=v0.265.0 capabilities=0c51f4f4ea45367838f06983af74ac3d9b535e2fcd115397145a654ab81b7c04 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
