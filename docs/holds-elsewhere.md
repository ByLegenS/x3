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

**A name inside a fence is an illustration.** A declared place may be a
procedure document, and a fenced block in one is usually a drawing — a directory
sketch, a layout, a listing. A bare name standing alone on such a line runs
nothing: it is redrawn when the tree changes, it never stops measuring. Counted
as a bond it locks a file the migration could have melted, and the pool stops
where the drawing is. The fence is **not** an amnesty: a path written the way a
shell runs it (`./gate.ps1`, `.\gate.ps1`, `/usr/local/bin/gate`) still holds
inside one, and so does every line carrying a declared selector or a running
step — the split is the shell's own rule, since a program in the current
directory cannot be started by its bare name. A name with words beside it
(`001_baseline.sql   frozen`) is still read as a bond: of the two ways to be
wrong, leaving a file in the tree is the cheap one. Measured on one repository's
twelve declared procedure documents: one such line, holding one exam file, while
all 34 lines that do run something begin with `./`, `.\` or an interpreter.

**A comment in a gate is not a step.** A gate script explains itself: why a file
is scanned, which defect was once measured, what the step used to be called.
Those sentences name files by name and run nothing — the same class as the
drawing above, in the language the script is written in. So a line is read as
its **code**: a full-line comment holds nothing, and a name written after a real
step (`go test ./src/ -run Parse   # was: src/legacy_test.go`) is not held
either, while the step's own subject still is. The file is read whole rather than
line by line, so a block comment is not mistaken for code on its second line, and
a language whose comment syntax is unknown is read as all code — a `#` heading in
a markdown procedure document is a heading, and still holds what it names.
Measured on a real production Go application: two exam files were held by nothing
but a plain comment (one Python `#`, one PowerShell `#`), and a third name stood
in the `SUSPECT` list for the same reason.

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

**The example is parsed, not scanned, and only a name it could really resolve
counts.** A word standing to the right of a selector (`r.Close()`, `json.Marshal`)
names a method or a field of some value, and the key of a composite literal
(`Report{Count: 2}`) names a field — neither mentions a top-level declaration.
A method the asked file declares cannot be meant by a bare word either: at the
call site a method is always tied to a receiver, and reaching it means naming the
type or the constructor that makes one, which is a top-level name and is caught.
Read as plain text instead, all three become bonds the moment a file somewhere in
the package happens to declare the same short word, and a migration then mints a
false `held` for every test file it empties — measured, on three counts in one
tree. An example whose body does not parse falls back to the plain reading:
a line that cannot be read is not evidence that nothing is bound to it.

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
