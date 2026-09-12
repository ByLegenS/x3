# A rule is declared where it applies

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 placement`

**What it catches:** the root configuration growing until nobody reads it.
Splitting it is already possible — that is the page above. What was missing is
that using the split was **nobody's obligation**, and an obligation with no gate
is a request. Measured on the pilot: one root settings file went from 824 to
1244 lines in a single sitting, **95% of the growth in the root**, and 16 of the
new rules named paths that all fell inside one module. Every one of them was
written by the person who had also written the rule against doing it.

```
x3 placement [-config <file>] [-out <file>] [-baseline <file>] [-update-baseline]
```

There is no directory argument. The subject of this gate is the configuration,
and the tree it is read against is the configuration's own directory — the same
base `include` patterns and `baseline.dir` already use.

### The law this gate keeps

> A declaration whose every named place falls inside **one** region must be
> written in that region's settings file. A declaration that weighs **two**
> regions belongs in the root file, and only there.

Regions are declared once, in the root file, as directory patterns:

```json
{ "placement": { "regions": ["apps/*"] } }
```

A pattern that matches no directory in the tree is red (`dead_region`): a region
nobody can be sent to is a rule that cannot be broken. A region is a directory
that holds at least one file the engine reads.

### Which region a declaration names

Every string under a declaration is read — values and keys alike — up to the
first segment carrying `*`, `?` or `[`. That literal head is what names a place:

| written | reads as |
|---|---|
| `apps/checkout/**` | inside `apps/checkout` |
| `apps/**` | above the regions, so it belongs to none of them |
| `**/*.go` | names no place at all, and decides nothing |
| `internal/core` | names no place either, while no region covers it |

So a rule is confined when the places it names resolve to exactly one region,
and unconfined when it names two, or names a directory that stands above them.
That is the whole answer to *"which rules legitimately stay at the root?"* — the
ones that compare regions, and the gate can see them because it counts.

### What is allowed to move

Only what the merge law can put back in the same place: **an object key** (parts
merge key by key, so any nested key can be written in a part) and **a whole list
item** (parts are appended, so half an item would not merge — it would become a
new rule). The gate therefore judges objects and list items, and never asks for
a field inside a list item. A named item keeps its name in the report
(`arch.rules[checkout-marks-its-payment]`); an unnamed one is identified by its
index, and an index moves when a neighbour is deleted.

Two kinds of section are never judged: this gate's own (its region list names
every region by definition) and **the engine's own** — the version pin,
`update`, `baseline`, `cache`, `adoption`. Those put no audit in force, so they
are not rules about a region, and a relative path written in them resolves
against the root file wherever it is written.

### A part's scope is its directory

Not a field inside the part. A part that declared its own scope would be an
exemption nobody had to justify: one line — `"scope": "/"` — and the gate is
closed for that file, quietly and forever. A directory is a fact on disk, and
widening it means moving the file. Nested regions resolve to the narrowest one.

The mirror holds too: a part reaching past its own region is red
(`declared_beyond_its_region`). A part speaks for one region, and a rule that
weighs two of them cannot be owned by either.

```
BLOCK x3.json: arch.rules[checkout-marks-its-payment]: declared_away_from_its_region
        every place it names is under "apps/checkout", but it is declared in
        x3.json; move it to apps/checkout/x3.json and name that file in "include"
        names: apps/checkout/**
```

### The exemption, and the run that judged nothing

An exemption carries a reason, and one that silences nothing is red — the law
every gate in the engine already keeps:

```json
{ "placement": { "regions": ["apps/*"], "allow": {
    "arch.rules[checkout-marks-its-payment]": "read next to its neighbour every time it changes" } } }
```

The exempted finding stays in the report with its reason and stops blocking; a
`policy` of `warn` does the same for every finding at once. Debt already in the
tree goes into the [baseline](baseline.md#the-finding-baseline) instead, which is where
adoption on a large configuration starts.

Every run prints how many declarations it looked at and how many named a region
at all, and a run where **nothing** named one is red (`nothing_measured`): a
gate that saw no subject can never go red, and the count is the only thing that
says so.

<!-- x3-dist version=v0.128.0 capabilities=796b04d7c3d74701288af9fa0577abc2b3913672a31190f52dd3e2d10c7a8e1e template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
