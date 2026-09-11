# Before a name is removed

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -holds`

**What it catches:** a criterion that keeps passing after the thing it measured
was deleted.

Criteria lean on names in the tree — a declaration, a path, the selector handed
to a runner. Remove one and a criterion can stop measuring without turning red:
a selector matching nothing makes most runners exit `0`, and `0` reads as
"done". A silent green is worse than a red one, because nobody looks at green.

```
x3 boxes -holds internal/parse/line_test.go
x3 boxes -holds-from removing.txt
```

Each entry is a **name or a path**, `-holds` comma separated and `-holds-from`
one per line (`#` opens a comment). An entry that names a file in the tree is
also asked as every top-level name that file **declares**: what gets removed is
usually a file, while the name a criterion holds is written inside it, and
leaving that step to a script outside the engine is what this mode exists to end.

**The form of the question does not change the answer.** `src/parse_test.go` and
`parse_test.go` are one question: a bare name is resolved against the tree, and
the report says which file it became (`resolved`). Asking both forms and merging
the answers by hand is a procedure, and a procedure is skipped by somebody, one
day. A bare name matching **several** files is answered for every one of them
and all are named - a guess would be silent when right and wrong when not.

**Nothing is run.** The question is not "does the criterion hold" but "does the
criterion lean on this name", and it has to be answerable **before** the
removal, not after it.

| `how` | The criterion | A search would |
|---|---|---|
| `text` | writes the name in its own words — `match`, `path`, `sources`, `query`, an argument | find the line, but not which box it belongs to, nor whether that box is open |
| `pattern` | hands a **selector** to a runner, and the selector read as a regular expression matches the name | **not find it at all**: a selector may be a fragment of the name |

Every record also says `via`: `file` when the criterion names the **file itself**
(by path or by bare name), `symbol` when it names something the file **declares**.
The two are not equally strong, and a run that treats them alike drowns the first
in the second - [Which hold is really a hold](holds-weight.md#which-hold-is-really-a-hold).

The second row is why the question is asked backwards, pattern against name
rather than name against text. A criterion running `-run ParseLine` proves
`TestParseLineKeepsBlanks`; searching the documents for that test's name returns
nothing. Which argument is the selector is **not guessed** — the kind declares
it (`markdown.criterion.kinds[].batch.select`), and no runner's flag name ever
enters the engine. Without that declaration the `pattern` row is silent, which
is the honest answer rather than a convenient one.

```
HOLD    WORK.md:197005faf068: pattern via symbol
	name: TestTheGatedWorkIsProven
	by: command go test -v ./... -run TheGatedWork
	box: the gated work a running test proves
	at: WORK.md:3
	asked: src/gated_test.go
x3 boxes: 1 asked, 2 name(s) - 1 held, 0 unsure, 0 free - 3 criterion(s) in *.md, 0 declared place(s), 0 suspect line(s)
```

**One field answers the question.** Every entry in `resolved` carries a
`verdict` — `held`, `unsure` or `free` — and a run prints a `FREE` line for each
free one. What may be removed is that list, read straight off the report.
Subtracting the record lists by hand is a rule somebody writes once, out of the
lists they happened to see that day, and the list they miss is the one holding a
real bond. `unsure` is **not** free: a record exists and could not be weighed.

Red when a name is held **or** when a record could not be weighed, green when
neither. The summary counts the criteria it read, so a green answer from a list
carrying **no** criteria can be told apart from a green that measured something.

<!-- x3-dist version=v0.99.0 capabilities=c87a76409332a713963f0bdac1bfd4b7896c2dae041df37989e6a12826e0e6d2 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
