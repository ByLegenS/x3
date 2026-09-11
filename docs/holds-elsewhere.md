# A name that lives outside the list

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -holds`, outside the list


A criterion is not the only thing that calls a test by name. A gate script does
too, and when the test is gone the script quietly runs nothing and still exits
`0`. Those places are **declared** — no engine can guess which file is a gate:

```json
{ "boxes": { "holds": { "sources": ["check.ps1", "gates/**/*.ps1", ".github/workflows/*.yml"] } } }
```

Every line is read as text, and two different things are found there. **The file
itself, written as a path**, is a bond — a person put it there — `USED`, `held`:

```
USED    gates/gate.ps1:4
	name: src/golden.txt
	by: Get-Content src/golden.txt
```

`places` says how many files were read, and **a declaration that reaches no file
stops the run**: an empty place answers every question with silence, and that
silence cannot be told from a declaration that died in a rename.

What the engine does with the **names and the places** on those lines is a
second question — [What a gate's line holds](holds-selectors.md#what-a-gates-line-holds).

**A mention in prose is not a place.** A document naming the test in a table cell
*recorded* something; it never *measured* it. Counting such a mention as a bond
was measured on a real production Go application and refused: of the 278 files no
criterion held, 206 were named somewhere in its documents, so the pool would have
fallen to 72 and the answer would have stopped carrying information. The record
still matters, but to a different gate: an [`arch consistency`](sets.md#two-sets-and-how-they-must-agree)
rule reading the names a document claims (`from: regex`) against the ones the
tree declares (`from: go`, `select: exported`) under
`compare: left-subset-of-right` is red on exactly that, and leaves this pool
untouched. On the same tree it named 144 claims out of 2330.

<!-- x3-dist version=v0.110.0 capabilities=8e148dba0bfdd51abe71c339e3f254364ebfc4f3e0ba2c90a6e0d9518d73398d template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
