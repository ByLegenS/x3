# A list of documents, carried to the machine

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -export`

**What it catches:** a migration that quietly drops what it could not translate.

```
x3 boxes -export <file> [-config <file>] [dir]
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
list. It would sit there either as a debt counted on every run or as a line
nobody reads; both turn a list back into a pile.

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

<!-- x3-dist version=v0.260.0 capabilities=8004fa03548574d60563f63f663d52673559fe0003df4bd6e444d702ee9ac59d template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
