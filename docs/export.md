# A list of documents, carried to the machine

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -export`

**What it catches:** a migration that quietly drops what it could not translate.

```
x3 boxes -export <file> [-export-deferred <file>] [-config <file>] [dir]
```

A project that kept its work as checkboxes inside documents
([how](boxes-document.md#a-list-written-as-a-document)) can carry them into the machine-written
list in one run. Nothing is measured, nothing is closed, and the only file
written is the one named:

```
x3 boxes -export work.yaml
x3 boxes -export: 603 box(es) written to work.yaml - 368 open, 235 done, read from 98 document(s)
x3 boxes -export: 355 box(es) stayed in the documents - 167 manual only, 184 with no criterion, 4 moved away
x3 boxes -export: 184 manual criterion(s) left behind, 73 criterion(s) lose the batch declaration
x3 boxes -export: 26 criterion(s) could not be carried; the list is written and this run is red
```

The kind words a document uses are that project's own (`pattern` may be written
as anything the `kinds` table declares). **They do not survive the crossing**:
the written list carries the engine's own `when` types and nothing else, so the
converted file reads the same in any repository.

### What stays behind, and what is a loss

Three numbers, and keeping them apart is the point of the command:

| | Meaning |
|---|---|
| **written** | boxes the machine can measure, in the new list |
| **stayed** | boxes left in the documents **on purpose** — a decision, not a loss |
| **could not be carried** | a criterion line the converter could not translate — a **loss**, and the run is red |

A box stays behind for one of three reasons: every criterion beside it is one a
person meets, no criterion is written beside it at all, or the item says the work
moved to another document and its criteria went with it. The first is the sharp
one and it is deliberate — **work a person does is not work the machine
measures**, and a box the machine cannot measure has no business in the machine's
*open* list. It would sit there either as a debt counted on every run or as a
line nobody reads; both turn a list back into a pile.

### The second destination: work a person measures

That reasoning names its own exception. A box the machine cannot measure has no
business in the list read on **every** run — but the deferred list is the list no
run reads unless it is asked to ([modes](modes.md#three-modes-of-a-work-list)), and that
is exactly where such work belongs. `-export-deferred` names it:

```
x3 boxes -export open.yaml -export-deferred later.yaml
x3 boxes -export: 1 box(es) written to open.yaml - 1 open, 0 done, read from 2 document(s) in *.md
x3 boxes -export: 1 box(es) written to later.yaml with the 1 manual criterion(s) beside them - no run reads that list unless it is asked with -deferred
x3 boxes -export: 0 box(es) stayed in the documents - 0 manual only, 0 with no criterion, 0 moved away
```

The box crosses **whole**: its manual criteria travel with it, unfiltered. Filtered,
it would arrive with no criterion at all and the loader would rightly refuse it.
A box with no criterion in the document still crosses nowhere — no mode of a
machine-written list can hold work it could never measure.

**One run writes both lists.** The second destination is a flag, not a
configuration field, and the difference is the whole design. A configuration that
reads its boxes from documents may not declare `deferred` — a document is read
either way, so a list kept inside documents has no modes, and that rule stays
exactly as it was. What the command *writes* is not inside a document; it is the
machine-written list itself, where modes already live. So the lock opens from the
command, and the configuration is untouched. Two separate runs are refused for
the same reason the two paths may not name one file: reading the same documents
twice can answer twice, and one box would then be counted in both lists.

**A deferred box may wait on somebody the open list may not.** A manual criterion
can name who meets it, and `manual.denyBy` refuses owners this list may not wait
on — an item waiting forever on a name nobody here can reach
([owners](boxes.md#who-a-manual-criterion-may-wait-on)). That ban does
not apply to the deferred list, and its own remedy says why: *move it to that
person's own list*. The deferred list is that — the list no run reads until it
asks. If the ban applied there too, the engine's own suggested destination would
be red on arrival, and deletion would be the only legal move left.

It is not silenced, it is **counted**. Every run that reads the deferred list
prints how many of its boxes wait on such an owner, because a deferral that grows
is a hiding place:

```
x3 boxes: 1 of them deferred, from later.yaml
x3 boxes: 1 deferred box(es) wait on an owner the open list may not wait on; the deferred list is allowed to, and this number says how often
```

Resume that box into the open list and the ban bites again, in the same run that
had nothing to say about it a moment earlier.

A loss is different, and it is never silent. Each one is printed with its box,
its place and its reason, and the command exits `1`:

```
BLOCK docs/PLAN.md:6c1f3a: box_record
	the criterion beside the box cannot be read: "nosuchkind" is not a kind this project declares
	at: docs/PLAN.md:12
```

The file is still written. The most expensive mistake a migration can make is not
a bad translation but an unreported one: a list that lost a quarter of its
criteria looks finished, and the source document gets deleted.

One loss the run reports but does not treat as a failure is the **batch
declaration**. A document's kind table can bind many criteria into a single run;
that binding is a property of the *kind*, not of the item, and the written list
has no place beside an item to keep it. The meaning is unchanged and the run is
slower, so it is counted and said out loud — otherwise the gate slowing down after
a migration reads as a regression nobody can explain.

<!-- x3-dist version=v0.263.0 capabilities=8d7619d885a6b4bc1ee14d49f384308ca33040428848fe899f93b0dbe3e77186 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
