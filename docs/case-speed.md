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
| every source file of every package it reaches, transitively, inside the module | an example calls a declaration, and that declaration calls others; a key that stopped at the package would show an old green after the code under it moved |
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

<!-- x3-dist version=v0.315.0 capabilities=8be0b8659b457bde304c60a4314857b93655a319461760dfcd9e7e01dcc7e225 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
