# What a change actually reaches

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 symbols`

```
x3 symbols [-out <file>] [dir]
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
-- no derived scope for "lane whatsapp · go vet": the symbol graph cannot be
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

**What it does not do.** It does not narrow non-Go steps, steps whose trials are
not all `go` commands, or steps that already observe what they read — those are
cheap and honest already. It does not follow a call graph inside a dependency: if
an application reaches one function of a package, every symbol that package uses
is in scope. That is the next granularity, not this one.

<!-- x3-dist version=v0.248.0 capabilities=dffebd956ea8894bdb647790ff20f0482f47e163ac3cdd5d0cc950e56f2fe723 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
