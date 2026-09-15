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

Three kinds of package are measured on **every** run, and the run says which and
why the first time it meets one:

- a package with a `testdata` directory beside it — the walk never enters it, so
  a fixture changing there changes no digest the key can see;
- a package whose sources declare `//go:embed` — what is embedded is not a Go
  source the walk reads;
- a tree with no readable `go.mod` at its root — without a module path an import
  path cannot be turned into a directory, and a key that cannot follow an import
  is a key that will eventually be wrong.

That list is the honest part. A cache is only worth having while the things it
cannot see are named out loud; `-no-cache` measures everything again and
`-cache <file>` points one run at another box.

The report names what it skipped in `cached`, and the summary line says
`N package(s) unchanged since the last run`. Findings and counts are identical
either way — the cache changes what a run **pays**, never what it **says**.

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

<!-- x3-dist version=v0.230.0 capabilities=763f5bce5b32f40c71c555bd608c5842ac33b94c161e541eddeb6df547e07ab5 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
