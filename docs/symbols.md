# What a change actually reaches

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 symbols`

```
x3 symbols [-doc] [-out <file>] [dir]
```

Every cache answers one question: *did my inputs change?* What separates a cheap
cache from a coarse one is **what it counts as an input**. A gate step declares a
file tree, so one line changed in a core package re-runs every step that named
that tree. Measured in a real production Go application: the core module exports
**2 088** symbols and one application uses **207** of them. A change to a random
core symbol reaches that application about **one time in ten** — and the gate runs
it every time.

⛔ **That is not a cache bug.** The cache is honest: it re-runs what it cannot
prove is unaffected. The fault is that it is never given the proof. This command
is the first half of the proof — for every declaration, a hash of what the
compiler would see:

```json
{ "symbol": "example/internal/gate.Run", "kind": "func",
  "file": "internal/gate/gate.go", "hash": "ce766842054a1312" }
```

**The hash weighs tokens, not text.** Indentation, line breaks, alignment and
narrating comments are what `gofmt` and every reader move around; none of them
change what a caller sees, so none of them are in the hash. Measured on this
engine's own tree — 3 664 symbols in 207 files, **0.2 s**:

| The edit | What the table did |
|---|---|
| a sentence rewritten in a doc comment | **byte-identical table** |
| a blank line and stray tabs inserted (`gofmt -l` now names the file) | **byte-identical table** |
| one line of one function body changed | **exactly one hash moved** |

**The baseline is the recorded table, not a git revision.** `git diff` was
measured against and rejected, and each reason is a failure mode a project would
otherwise inherit: the base is ambiguous (`HEAD~1`? the last deployment? `main`?)
and a rebase silently picks the wrong one; only committed work is seen, so a
dirty working tree measures something else; a `gofmt` run, a moved function or a
renamed file are all reported as change when nothing changed. Git may still serve
as a fast pre-filter for *which files are worth parsing* — the table has the last
word.

**A directive is not a comment.** `//go:embed`, `//go:linkname` and their kin
change behaviour, so they are part of the hash; only the narrating comment is
dropped. A symbol declared twice under exclusive build tags keeps **both** halves
of its hash: choosing one would be choosing not to see the other change.

**A file that cannot be parsed is named, loudly.** Its symbols are not in the
table, and a table that shrinks silently hands a wrong green to every gate that
reads it. The summary counts it and the run prints `UNPARSABLE <file>`.

### Who uses a symbol — the reverse graph

```
x3 symbols -graph [-out <file>] [dir]
x3 symbols -uses <package.Symbol> [dir]
```

`-graph` type-checks the module and inverts `types.Info.Uses` into **symbol → the
packages and files that use it**. Measured in a real production Go application:
242 packages, 7 694 used symbols, 10 690 edges, 138 interfaces, **2.4 s**.

That measurement is also the case for doing it at all. Of the core module's 5 605
used symbols, **84% reach no application at all**; one application uses 363 of
them and another 660. A change to a random core symbol therefore reaches a given
application about one time in fifteen — and today the gate runs it every time.

⛔ **Import edges would have thrown the win away.** The same application imports
73 of the core's 89 packages while using 207 of its 2 088 symbols. Package
granularity answers *"is it connected?"*; the question a cache needs answered is
*"did the thing I use change?"*

⛔ **An interface is an edge, and it is the one that hides.** Code calling
`Sender.Send` never writes the name of the concrete method it reaches. Every
named type in the module is asked `types.Implements` against every interface in
it (pointer receivers included), and the interface method's users are bound to
the implementation's. Without that edge, changing an implementation passes
unmeasured while its caller still compiles — and then behaves differently.

⛔ **Uncertainty is named, never assumed away.** A file importing `reflect` or
`plugin`, or carrying a `//go:linkname` directive, hides an edge the type checker
cannot see. Such a file is printed `UNDETERMINED <file>: uses <what>` and widens
to *everything*. **A missed edge is a wrong green, and a wrong green is worse
than a slow gate** — this engine learned that once, when the cache was losing
findings. The directive is recognised where it is *written*, not where it is
*mentioned*: searching the text was measured and produced a false positive on the
very file that names the directive in a constant.

**A workspace is loaded whole.** Under `go.work`, `./...` alone returns only the
root module's packages; on a repository whose core is a separate module that
silently loses the half of the graph the question is about (measured: 43 packages
of 242). Every `use` entry is loaded by name, and a `go.work` that cannot be read
is an error rather than a quiet fall back to the root.

**A package that does not type-check stops the graph.** The run says so and exits
`2`. A partial graph would report "nothing is affected" for every edge it failed
to see — which is the one answer this capability must never give by accident.

### What moved since the recorded table

```
x3 symbols -since <recorded table> [-out <file>] [dir]
```

The run compares today's table with a recorded one and names
**changed, added and removed symbols**. There is no baseline on a first run, and that is not an error:
the run says the table is not there yet and writes one.

The three controls below are the argument for the whole design, measured on this
engine's own tree:

| The edit | What was named |
|---|---|
| one line of one function body | `CHANGED` — **one symbol** |
| the file renamed (`git mv`), not a byte of code touched | **nothing at all**; `git diff` would have called it a delete plus an add and made every symbol in it new |
| a function renamed | **one added, one removed** — plus the one caller whose body now names something else. Not "the whole file changed" |

### The scope a gate step actually rests on

```
x3 symbols -scope <target>[,<target>...] [-out <file>] [dir]
```

A gate step that declares `touches: ["<app>/**", "<core>/**"]` re-runs on every
line of the core, because that is what the declaration says. A step whose trials
are all `go build|vet|test|run|list` commands naming their targets gets its scope
**derived from the graph instead**: the symbols the target's own packages declare,
plus the ones its transitive dependencies actually use. The cache then records
those symbols and their hashes in place of the file trees.

Measured on the same real production Go application:

| Target | Rests on | Of the tree |
|---|---:|---:|
| one application | 5 021 symbols | 56% |
| another | 5 973 | 67% |
| a service | 2 905 | **32%** |

**The control experiment is one file with two functions in it.** One is inside
an application's derived scope, the other is not — a file-level scope cannot tell
them apart:

| Change to `<core>/action/kind.go` | The application's `go vet` step |
|---|---|
| a function the application never reaches | **skipped**, 0 steps ran |
| a function it does reach (indirectly) | **ran**, and said why: `ran again: symbol <core>/action.Alias changed` |

Across the whole gate, one such out-of-scope core change: **54 steps ran before,
40 after**, 254 s of serial work down to 194 s — with **the same 21 reds, named
identically**. A faster gate that loses one finding is not faster, it is broken.

⛔ **A scope that cannot be justified is not narrowed.** If the graph cannot be
built the step keeps its declared trees and the run says so by name:

```
-- no derived scope for "lane store · go vet": the symbol graph cannot be
   built: 2 package(s) do not type-check, first is <core>/action: ...
   ; the declared trees stand
```

Measured by making one file un-parseable: every affected step reported the reason
and ran. The same refusal covers a target that names no package, a trial running
in a fixture tree, and a package in the closure whose edges cannot be seen.

**The record holds two digests, not the symbols.** A step's memory carries the
digest of the whole table and the digest of its own scope. That ordering is the
cheap path: if the table's digest is unchanged, no declaration and **no import
line** moved, so no step's scope can have moved either — and the type checker is
never run. It is asked only when the table did move, and then once for the whole
run. Measured on the same repository, nothing changed between runs:

| | symbols written into the record | two digests |
|---|---:|---:|
| cache file | 22 MB | **4.2 MB** |
| warm run, wall clock | 14.4 s | **7.4 s** |

⛔ **An import line is an input.** A blank import (`_ "x"`) runs `init()` without
changing a single declaration, so a table that weighed only declarations would
call that run identical. The import list — aliases included — is weighed with
them. Control: adding `_ "embed"` to a file and touching nothing else moves
exactly one entry, `<package>.imports`.

⛔ **An embedded file is an input.** A `//go:embed` directive keeps its text when
the page it embeds changes, and the binary serves a different page. The hash of
the declaration carrying the directive therefore weighs the **content** of every
file its patterns match (a directory pattern counts dot and underscore files
too, and a pattern that matches nothing is weighed by its name). Measured in a
real production repository: a page embedded by the server a `go run` step drives
gained an element 3 200 px wide, and the step came back **green from the cache**;
an unrelated comment in the gate configuration made it run, and the same page was
**red**. Control experiment `cache inner run and embedded file control
experiment`: the embedded page moves and no declaration does — the old engine
answers `0 ran, 1 skipped`, this one runs the step and catches the wide page;
going back to the first page is still a hit.

⛔ **An inner run's observation is not the whole step.** When a trial's own
command is not `x3` (`go run ./cmd/<tool>`, a shell tool) but an `x3` runs inside
it, that inner run reports what **it** read — its configuration — and nothing the
outer command opened. Such a step's record is the **union**: the observation,
plus the derived scope when there is one, plus the `touches` the step writes
itself (and the inherited scope when nothing can be derived). With neither, the
step is not remembered and runs every time — a slow gate is visible, a blind one
is not. Same control experiment: a wrapper reads the page it declares in
`touches` while its inner run reports only `x3.yaml`; the old engine answers
`0 ran, 1 skipped` on the wide page, this one runs it and is red.

**What it does not do.** It does not narrow non-Go steps, steps whose trials are
not all `go` commands, or steps whose trials are all `x3` itself (those observe
what they read and are cheap and honest already). A file a command opens because
an **argument** names it (`-page <file>`) is not derived; declare it in
`touches`. It does not follow a call graph inside a dependency: if
an application reaches one function of a package, every symbol that package uses
is in scope. That is the next granularity, not this one.

### What each symbol is for — `x3 symbols -doc`

A capability catalogue written by hand goes stale the week it is written.
Measured in a production repository: its hand catalogue said one interface had
40 methods, the code had 36, and it named a command directory that no longer
existed. The only part of a catalogue a person has to write is the sentence that
says what a thing is for — and that sentence already sits in the code, above the
declaration, where it changes in the same commit as the code.

`x3 symbols -doc` adds that sentence to the table. Every **exported** symbol
gets an `about` block, every package its doc paragraph:

```yaml
symbols:
  - symbol: example.com/shop.Cart.Add
    kind: method
    file: shop.go
    hash: 3f0c...
    exported: true
    about:
      doc: Add puts one line in the cart and says how many there are.
      receiver: Cart
      result: int
      methods: []
packages:
  - path: example.com/shop
    doc: Package shop sells things. It keeps no stock of its own.
    file: shop.go
```

| Field | What it holds |
|---|---|
| `about.doc` | the **first sentence** of the doc comment (cut at the first period followed by a space; a single capital before it is an initial, not an end). Directive lines (`//go:`, `//x3:case:`) are not prose and are dropped. An undocumented symbol has `doc: ""` |
| `about.receiver` | a method's receiver type, without `*` or type parameters |
| `about.result` | a function's result types, names dropped: `int`, `(int64, error)`; empty when it returns nothing |
| `about.methods` | an exported interface's exported methods, each with `name`, `doc`, `result`, in written order. An embedded interface is **not** expanded |
| `packages[].doc` | the first paragraph of the package doc comment; the file with the smallest path wins when several carry one |

**Every key is written on every exported symbol, empty or not.** A template treats
a missing key as an error; an undocumented symbol or a function with no result is
information, not a fault, and must not need a guard in every template.

**Exported means visible to an importer.** A name in a `_test.go` file, an
unexported name, and a method of an unexported type carry no `about` block.

**Without `-doc` the table is byte-identical to what it was.** Measured on this
engine's tree with the engine before and after the flag existed: same 766 178
bytes. `-doc` together with `-graph`, `-uses` or `-scope` is refused (exit 2) —
it adds to the table, and those modes do not print the table.

**A catalogue is an `emit` document whose source is the table.** `source:
symbols` runs the documented table on the emit root; `source: symbols <dir>`
runs it on a subdirectory with its own `go.mod` (a directory outside the root is
refused). The engine brings the measurement, the project brings the shape:

```yaml
emit:
  documents:
    - name: catalog
      out: CATALOG.md
      template: catalog.tmpl
      source: symbols
```

```
{{range .Report.packages}}## {{.path}}: {{.doc}}
{{end}}{{range filter "exported" "true" .Report.symbols}}- {{.symbol}} ({{.kind}}{{with .about.receiver}}, on {{.}}{{end}}{{with .about.result}}, gives {{.}}{{end}}): {{or .about.doc "UNDOCUMENTED"}}
{{range .about.methods}}  - {{.name}}{{with .result}} gives {{.}}{{end}}: {{or .doc "UNDOCUMENTED"}}
{{end}}{{end -}}
```

`x3 emit -check` then keeps the catalogue honest: edit one doc sentence in the
code and the catalogue is `stale_document`; regenerate it and it is green.
Control experiment: `symbols doc catalogue control experiment` in `x3.yaml` —
the fresh arm weighs every line of the catalogue (first sentence, receiver,
result, interface method, undocumented symbol, absent test and unexported names),
the edited arm is red, the regenerated arm green. Breaking the sentence cut in the
engine turns the fresh arm red.

**No per-file cache, on purpose.** Measured on this engine's 260 files: the
documented table takes 0.21 s; a per-file cache of the parse result, read back
warm, took 0.38 s — decoding the stored result costs more than Go's parser.
A catalogue that has not changed is not rebuilt anyway: the gate's step cache
skips the `emit -check` step when its declared tree did not move.

<!-- x3-dist version=v0.337.0 capabilities=3860e842c699cce7f98e5bd335013a1d4b84da590e84edf414492455d8caaf11 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
