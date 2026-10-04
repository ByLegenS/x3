# What a run costs, and what it does not pay twice

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, what a run costs

An example is a test the engine writes, and a test costs a **process**. Measured
on this engine and on a real Go application of 3 649 examples in 94 packages,
the process was almost all of it: an empty `go test` in one of that
application's packages is 0.63 s, and the same package asked by this gate is
0.56 s — the examples themselves cost nothing measurable. Ninety-four packages,
one after another, is a minute and a half of starting processes.

Two things follow, and both are below: packages do not have to wait for each
other, and a package nothing has changed does not have to be asked at all.

### Packages that run beside each other

Every package is its own compilation and its own test binary, so nothing makes
them queue:

```json
{ "case": { "workers": 8 } }
```

Default: **half the processors**, and `-workers <n>` moves a single run. The
ceiling is not a preference but a measurement made on the `mutate` lane, where
the same question was asked first: every worker starts a compiler, and a run
that filled every core left the machine unusable — 55 processes, the processor
at 96 °C. Time is the machine's idle hours; heat is the hardware's life. A run
that wants the whole machine says so.

**The report does not know how many ran.** A finding is written onto the example
that owns it and the report is built afterwards, in tree order, so one worker
and eight produce the same findings in the same places. The control experiment
asks exactly that.

### A package the run does not run again

```json
{ "cache": { "dir": ".x3cache" } }
```

The same directory the other caches use, under the command's own file. An entry
is used only when **everything the package's answer depends on** is unchanged:

| What the key carries | Why it is in there |
|---|---|
| every source file of the package | the obvious half |
| of every package it imports, transitively, inside the module: **the declarations it can reach**, plus every file's package clause and imports | an example calls a declaration, and that declaration calls others; a key that stopped at the package would show an old green after the code under it moved |
| the build tags the run carries | the same tree answers differently under `-tags` |
| the ceiling, the report version, the Go toolchain version | each of them can change the answer without changing a source file |
| `go.mod` and `go.sum` | a dependency moving is a change nothing in the tree records |
| the engine version and a fingerprint of the whole configuration | the same three the other caches use |

**Only a green package is stored.** A stored red would repeat, on a later run, a
sentence that run never measured — and the work it names is already being fixed,
so measuring it again costs nothing anyone minds. A package with a **deferred**
example is not stored either: nothing measured it.

Two kinds of package are measured on **every** run, and the run says which and
why the first time it meets one:

- a package whose `testdata` directory cannot be read — the key cannot weigh a
  fixture it cannot open;
- a tree with no readable `go.mod` at its root — without a module path an import
  path cannot be turned into a directory, and a key that cannot follow an import
  is a key that will eventually be wrong.

⛔ **A fixture is part of the package.** The source walk never enters `testdata`,
so the key weighs that tree on its own: every file by content, every directory
by name. A package carrying `testdata` used to be measured on every run, and so
was every package importing it — measured on a pilot, one `testdata` kept 25
packages running with nothing changed (48 s). Experiment `case testdata package
memory control experiment`: the second cold-to-warm arm is red on v0.313.0; a
file joining the fixture tree, and a fixture changing, run the package and its
importer again.

⛔ **An embedded file is part of the package.** A package whose sources declare
`//go:embed` used to be measured on every run, and so was every package importing
it — measured on a pilot, the record held 30 of 119 packages and `x3 case` took
306 s. The key now weighs the **content** of every file the patterns match, read
the way the symbol table reads them (a directory pattern counts dot and underscore
files too, an indented directive inside `var (` counts, a pattern matching nothing
is weighed by its name). Experiment `case embedded package memory control
experiment`: nothing changed → `3 package(s) unchanged` (the old engine: `1`); a
file joins the embedded directory → the embedding package and its importer run
again; the page changes → both run and their examples catch it.

⛔ **An imported package is weighed by what the importer can reach, not by its
whole text.** Measured on a pilot (v0.319.0): `return x` → `return (x)` in one
core function re-ran every importer, the core region 189 s with case 185 s,
though most of them never reached that function. The key now reads each imported
package as declarations. The seeds are every `q.Name` the importing package
writes — in its code, its tests and its example lines. A reached declaration
reaches the names it uses, and a reached **type** reaches **every method** it has.
That is why an interface call needs no implementor search: a value's dynamic type
is born in code that names it, so whatever calls its method — an interface,
reflection, `fmt` asking for `String` — calls a method already in the key. Every
package-level `var` and every `init` runs at start-up and is always reached. A
package is weighed whole wherever reach cannot be read: `//go:linkname`, `import
"C"`, a function without a body, a dot import, a file that does not scan. A blank
or dot import now joins the importer's key too (before, `import _ "m/x"` left x
out of it).

```text
lib.Unused changes      → use and idle recalled
lib.Double changes      → use runs (it calls Double), idle recalled
square.Area changes     → use runs (it calls it through lib.Shape), idle recalled
var Base = 10 → 11      → use and idle run (a package initialiser runs in both)
```

Experiment `case symbol-level reach control experiment`: the five trees under
`internal/cases/testdata/reach` differ only in `lib/lib.go`; the stamp and
cache-backed `Check` arms beside `stamp` and `Check` assert each row, and with
the narrowing switched off the four "recalled" arms turn red. A compile error in a
declaration nobody reaches leaves the importer recalled; the red comes from the
package itself (its own examples, or the build step).

**Inside the changed package, each example is remembered on its own.** When a
package's key misses, every example is asked again by its own key: what lies
outside the package (as above), its own text and its file's `//x3:` lines, the
package's test, fixture and embedded files whole, every file's package clause and
imports, and only the declarations **this example reaches** - the names it
writes, followed through the package; every `var`, `init` and `TestMain` is
always reached, a reached type takes every method. An example whose key matches
its last green is recalled; the report lists it under `recalled`.

```
p.double changes        -> Twice runs (it calls double), Next/Scaled/Same recalled
var factor = 1 -> 2     -> all four run (a package initialiser)
the fixture changes     -> all four run (fixture files are weighed whole)
a type error in idle()  -> one example still runs, and the build red is seen
a comment changes       -> one example runs as a witness, three recalled
```

A package that cannot be narrowed (cgo, linkname, assembly, dot import, a file
that does not scan) or an example that reaches a body naming `"go"` or
`.Executable` (it may run the package outside its test binary) runs whole. When
the project declares a test database template, its name - the digest of the
migrations - joins every key: a schema change runs every package again.
Experiment `case example-level reach control experiment`: the six trees under
`internal/cases/testdata/inner` differ in one line each; the cache-backed arms
beside `Check` assert each row, and with the witness switched off the type-error
arm turns red.

**An opt-in file nobody asked for does not keep its package out of the cache.**
Its examples are counted under `optin` and are not this run's question, so the
package's green is stored for the rest (before, such a package ran on every
unchanged run). When the run carries the opt-in tag (`-tags live`) the package
key is not asked: the opt-in examples run every time they are asked for and are
never recalled, while their siblings go through the per-example memory. The
run's tags join every key, so the first asked run after unasked ones measures
the siblings again; the next asked run recalls them.

```
opt-in file, nothing changed      -> cached: "cache 1 hit(s), 0 miss(es)"
double changes, live not asked    -> Twice runs and its red is seen, Next recalled
live asked, second run            -> Probe runs, Twice and Next recalled
live asked, Probe changes         -> Probe runs and its red is seen, siblings recalled
```

**Why did a package run? `X3_CACHE_WHY=1`** prints one stderr line per package
the run measured, and it reaches a gate step through the environment:

```
x3 case: why lib: whole because a.go uses cgo
x3 case: why app/core: 1775 of 1775 example(s) run: 1775 reach a changed declaration
x3 case: why p: 2 of 3 example(s) run: 1 reach a changed declaration; 1 are opt-in and asked for
x3 case: why q: 4 of 4 example(s) run: 4 cannot be narrowed (it reaches Build, whose body may start a subprocess)
```

The other counts are `have no record` (first run, or the record was written
under other tags) and `1 runs as the witness that the package still builds`.
Experiment `case second unchanged run misses nothing`: the same tree under
`internal/cases/testdata/inner/optbase` twice on one cache; with the old store
rule (any missing tag refuses) the second run says `1 hit(s), 1 miss(es)`. The
four `opt*` arms beside `Check` assert the rows above; with the old rule the
unchanged arm and both asked arms turn red.

That list is the honest part. A cache is only worth having while the things it
cannot see are named out loud; `-no-cache` measures everything again and
`-cache <file>` points one run at another box.

The report names what it skipped in `cached`, and the summary line says
`N package(s) unchanged since the last run`. Findings and counts are identical
either way — the cache changes what a run **pays**, never what it **says**.

The record of `x3 case` is bound to the `case` and `testdb` sections of the
settings only, not to the whole configuration. A rule's text in another section,
or another working copy of the same repository sharing the cache directory with
a different gate, no longer throws every package away:

```text
x3 case: 3 example(s) in 3 package(s) - 3 passed, 3 package(s) unchanged since the last run, 0 finding(s)
```

Measured on a pilot before this: the record held 30 of 119 packages, because
each run under a different whole-configuration salt kept only what it had just
run. A change to a file a package reaches still runs that package (and every
package importing it); a change to the `case` section itself still runs them all
(experiment `case package memory settings section control experiment`).

The per-file caches follow the same rule: `comments`, `secrets`, `lang`
(`language`), `scan` (`expect`, `scan`) and `test` (`test`, `testdb`) are salted
by the sections they read, so editing another section keeps every file's record:

```text
x3 comments: cache 2 hit(s), 0 miss(es)
```

Before this a change anywhere in the configuration measured every file again
(pilot: 24-27 s for one step). Experiment `per-file salt control experiment`:
a cold run misses both files, a change in `secrets` hits both, and the control
arm (a change in the `comments` section) misses both again.

A cache file is always written so it reads back: multi-line text (an indented
finding, a tool's output) is stored as one quoted line, the file is decoded
before it replaces the old one, and a writer that finds the file held open by a
reader retries instead of losing its slot. A file that cannot be read is an
empty cache — the run is cold, never red — and a reader of the record says
`<file> cannot be read ..., so it remembers nothing` instead of a decoder error.

A gate trial that could not run at all (its tool cannot be started: exit
`-1`), or whose output carries the engine's own `this run cannot measure`, is
red for this run only: the line says `environment error, not cached` and the
next run measures the step again. A red the trial itself decided stays cached.

```yaml
trials:
  - run: [some-tool-this-machine-lacks]
    want: 0     # red now: "environment error, not cached"; measured again next run
```

### What each example costs

The report carries the ten slowest examples it measured:

```json
{ "slowest": [ { "file": "wallet.go", "line": 8, "target": "Add", "seconds": 1.204 } ] }
```

This is what makes the ceiling sentence useful. Examples inside one package run
one after another, so when the package hits `case.timeout` the one holding the
clock is simply whichever was running — accusing it would accuse an innocent,
and the gate still refuses to. But it no longer says the slow one cannot be
known: the finding now names the slowest examples it **did** measure, which is
where the reader has to look.

### The work a run leaves behind

Nothing is ever written into the measured tree. The generated tests go to a
directory under the system temp directory whose **name is the hash of what is
in it**, and they are written through a second name that is renamed into place
when the last byte is down — so a half-written directory never carries the real
name, and two runs producing the same tests share one directory instead of
racing over it.

That is also the answer to a leak. The old run made a random directory and
removed it with a `defer`, which a killed process never reaches: measured, one
of them lived six hours and was deleted by hand. Content-addressed directories
repeat instead of accumulating, and anything under that root nothing has touched
for twelve hours is swept at the start of the next run — the engine's own litter,
collected by name.

⛔ **`-count=1` stays on the toolchain call.** Go's test cache is not this
gate's cache: it keys on the package's own files and knows nothing about the
question being asked — the configuration, the carried tags, the ceiling. The
decision not to run belongs to the gate, which can see all three; a package that
reaches the toolchain has reached it **to be measured**.

### Measured

| `x3 case` on that application | Time | What ran |
|---|---|---|
| one package at a time, no cache — what the step cost before | **117.1 s** | 94 packages, 3 649 examples |
| packages beside each other, no cache | **22.2 s** | the same 94 |
| the first run with a cache — it fills it | 22.2 s | the same 94 |
| the same tree asked again | **17.1 s** | 33 skipped, 61 measured |

All four produced the **same findings and the same summary**; they differ in what
they paid, never in what they said. The second row is the whole of `workers`.

The fourth row is the cache, and it is the honest one: only 33 of 94
packages were skipped, because that application's gate is red on 42 packages
today and **a red package is measured again every time** — plus the packages
whose directories carry a `testdata` never enter the cache at all. On a green
tree the last row keeps shrinking; on a red one the cache pays for the part that
is already green and nothing else. Measured on this engine's own tree, where 9 of
12 packages carry `testdata`: 11.1 s one at a time, **3.0 s** beside each other,
and the cache changes nothing it is allowed to change.

<!-- x3-dist version=v0.322.0 capabilities=5db2a1d21099dbaeb144dc1fc7e191496f0cc6161a3b825ecbd87a7d13c9a11d template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
