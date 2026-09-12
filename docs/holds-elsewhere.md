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

### The engine's own example is a third place, and it is not declared

A production file can carry an inline example, and that example's setup runs
**real code**:

```go
//x3:case: given=(n := seedLedger()) in=(n) out=7
func Total(n int) int { return n }
```

`seedLedger` lives in a test file that carries no exam of its own, so no
criterion names it and no gate script calls it. Asked, the pool answered
**`free`** — and deleting it does not lose a measurement, it stops the package
compiling: `x3 case` then prints `undefined: seedLedger`, `does_not_build`.
This blindness **grows** with a migration, because every round turns one more
test file into a helper-only file.

Nothing is declared for this one: the directive is the engine's own, so the
engine reads it. The bond is reported as `USED`, `how: "example"`, `held`, and
`examples` in the summary says how many directives were read.

**The bond is the package, not the text.** The generated exam is written into
the directory of the file carrying the directive, so an unqualified name
resolves only there. A namesake in another package is **not** held — read as a
text search instead, the pool would fill with homonyms. A name an outsider could
write (exported, package-level) that appears in an example elsewhere is
`SUSPECT`, `unsure`: a qualified call spells the same word.

<!-- x3-dist version=v0.128.0 capabilities=796b04d7c3d74701288af9fa0577abc2b3913672a31190f52dd3e2d10c7a8e1e template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
