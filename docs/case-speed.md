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

### One package's examples split across processes

`workers` spreads *packages*; inside one package the examples used to run one
after another in a single test binary. A body change that reruns 1 512 examples
of one package therefore took as long as the slowest serial chain — 23-66 s on
the pilot, the binary alone 60 s. Now the run compiles that binary **once** and
`go test -exec` hands it back to x3, which opens it in as many processes as the
package's machine slots allow (`machine.yaml`, shared across x3 processes), at
least 24 examples a part. Each process runs only its own examples; their output
is written part by part, the exit code is the worst one, so a red example or a
panic in any part stays red.

```json
{ "case": { "shards": 4 } }
```

`0` (the default) splits as wide as the slots — but parts that each need a
database copy stop at **four**; `1` never splits; `-shards <n>` moves one run
(and may open more copies). The number is not part of the cache key: splitting
changes the time, never the verdict. Why four, measured on the pilot (480
database examples, 32 slots, one body change):

| parts | binary | whole run |
|---:|---:|---:|
| 1 | 11.6 s | 20.2 s |
| 4 | 5.0 s | 14.1 s |
| 8 | 4.6 s | 17.2 s |
| 20 | 11.5 s (19 copies took 9 s, examples 3× slower) | 22.0 s |

Past four the server, not the processors, is the bottleneck.

**The plan weighs time, not count.** A count said nothing about weight: on the
pilot a region run after one body change reran 92 packages in 95 s, and the
step ended when the heaviest package (762 examples, about 80 s of work) did —
it started late, in tree order, after small packages had taken the three
shared copy seats, so it ran in two parts. A nine-example package carried 35 s
(one example alone 18.7 s), a 140-example one 74 s. The run now keeps each
package's example durations in its cache (`<dir>#took`) and plans from them:

- the queue runs **heaviest first** (the widest package still leads, alone);
- the copy budget — a quarter of the machine share, at least three, or
  `case.shards - 1` — goes part by part to whichever package then has the
  **longest work per part**, never below 2 s of work a part, never more than
  eight copy parts a package (8 parts 4.6 s against 20 parts 11.5 s above);
- slots are dealt to parts **by duration**, longest first into the lightest
  part, so an 18 s example does not share a part with the rest.

An example with no recorded duration weighs its package's known average, or —
with none — what the old rule implied (24 examples a part). Durations shape
only the plan: which example runs, its verdict and its records are unchanged.
The control experiment (`case split control experiment`): sixty examples with
no record are dealt into two parts and the one wrong example is red with its
own sentence, the other 59 pass once each; the second run, durations recorded
and the package lighter than one part, runs the same red unsplit — the engine
before this change split it again. Measured on the pilot only up to the
breakdown above; the region run after this change is not measured yet.

**A database is never shared.** Two parts writing the same database see each
other's rows. A package is split only when every part can take its own copy
(`testdb` with `case.database: isolated`, or a run inside `x3 testdb run`: the
first part keeps the run's copy, the others get fresh ones, made while the
binary compiles), or when the tree declares no `testdb` at all. A tree that
declares one but runs on the shell's database is not split. A part whose copy
cannot be made is folded back into the others.

**Copies leave with the outer run.** On Windows every `DROP DATABASE` waits for
a checkpoint: dropping 31 copies one after another took 77 s, three side by
side 4.1 s, and the outer `testdb run` then spent another two dropping its own.
`testdb run` now hands the command a file (`X3_TESTDB_ADOPT`); a copy made
inside is written there instead of dropped, and the outer run drops all of
them side by side with its own, in the same wait. Outside a `testdb run` the
copies are dropped side by side before `x3 case` returns — never left behind
(a run that exited without waiting was measured leaving 4 of 8 copies on the
server).

```
$ x3 case -only internal/cases -no-cache -shards 1     # x3's own tree, 243 examples
x3 case: 243 of 2213 example(s) selected - 243 passed ... 0 finding(s)   13.6 s
$ x3 case -only internal/cases -no-cache               # split
x3 case: 243 of 2213 example(s) selected - 243 passed ... 0 finding(s)    6.8 s
```

The generated test reads its part's slots from a file named by `X3_CASE_SLOTS`
and **unsets it** before any example runs: an example that starts an x3 run of
its own (x3's own tree does) would otherwise filter the inner run by the outer
part's list — measured, 11 false reds. Go's test log and the coverage record get
one name per part and are merged before the binary returns, so what the
examples opened and reached is recorded as before. An example that only passes
after a neighbour wrote a package variable may pass alone and fail split; that
was already a hidden order dependency, and the split makes it visible.

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

⛔ **A file an example opens at run time is part of the package.** A test binary
writes every path it opens or stats to Go's own test log (`-test.testlogfile`,
the log Go's test cache checks with); `x3 case` passes that flag and reads the
log after the run. The paths inside the measured tree are kept with their
content (a directory by its names, a missing path as absent) **in the package
record and in the example record**, and the next run checks them before it
trusts either: one moved file runs the package again, every example of it,
since the log belongs to the process and cannot tell which example opened what.
The same paths reach the gate's observation of the step (a file by its whole
bytes, a directory and a missing path by name), so a gate step running `x3 case`
measures again when such a file changes. Measured on a pilot: an approved
manifest an example read with `os.ReadFile` changed, the `case` step returned
its old red as `cached: nothing this step reads has changed`, and only `-force`
showed the green. Limits: a path outside the measured tree is not weighed; a
path the run itself wrote (changed after the run began) is output, not input;
a file a child process opens is not in the log. Experiment `case run-time read
control experiment`: the example reads `rate.txt` beside it; the changed limb
moves only that file and the package runs again and is red (the old engine:
`cache 1 hit(s)`, green), the file back is one hit again, and the gate step on
the changed limb runs and is red (the old engine: `0 ran, 1 skipped`).

⛔ **A cached region run stays under 30 seconds.** Measured on a pilot (v0.348.0 →
this release, same tree, no `-force`): nothing changed — a region 19 s → 8 s; one
line added to a box file — the core region 54 s → 26 s; a commit touching only a
box body — the messaging region 70 s → 30 s. Five causes, each removed without
changing what is measured:

- the walk that enters hidden directories (`hidden: true`, architecture rules)
  never enters the version-control directory (`.git`, `.hg`, `.svn`, `.bzr`,
  `.jj`): every commit wrote new objects there and a 54 s step ran again on a
  commit that changed one box line;
- validation weighs a record's files and directories in parallel, with one ignore
  filter per run (it was rebuilt for every directory: 42 910 of them);
- the snapshot (`x3 docs`, `x3 snapshot`) decodes only step records — trial
  recalls were 450 MB of a 671 MB record and decoded to nothing;
- a trial that only reads a `{tmp}` file another trial wrote (`-from`) runs
  beside the other readers once the writer is done:

```yaml
trials:
  - run: x3 boxes -config x3.yaml -out {tmp}/r.yaml .       # writer: runs alone
  - run: x3 emit -check -from evolution={tmp}/r.yaml         # readers: side by side
  - run: x3 emit -check -only experiment -from experiment={tmp}/r.yaml
```

- when validation leaves only a few steps to run, the processor budget of the idle
  workers goes to them (workers × cores is never exceeded; the share is not part of
  any key).

Control experiment on the pilot: an untracked file added under one application
directory runs the architecture step again and the unused-file step turns red;
the file removed, the same region is `0 ran, 116 skipped` from the cache in 10 s.
A commit that only adds `.git/objects` runs the architecture step no more.

⛔ **A file a package embeds is told to the gate too.** The case record has always
weighed `//go:embed` files, for the embedding package and for everything importing
it; the gate's record of the step had not heard of them, because no read of the
run names them. Measured on a pilot where a design arrives as a declaration file
in an embedded directory: one line removed from it, and `x3 case` behind a gate
step said `cached: nothing this step reads has changed`, exit 0, while `--force`
was red. The step now holds, by their whole bytes, the files embedded by the
packages that carry an example and by what those import; the case record still
decides which examples run, so a template edit re-runs the examples that reach it
and not the tree.

⛔ **A working copy kept inside the tree is not this tree's input.** The log of what
an example opened stops at a directory carrying its own `.git`. Measured on a
pilot: one example that walks the repository wrote old working copies kept under a
hidden directory into the step's record, 11,400 files and 30,600 directories, and a
step that ran nothing took 6.6 s instead of 0.3 s to check itself.

Experiment `case embedded file control experiment`:
one embedded byte changes and the gate step runs and is red (the old engine: `0
ran, 1 skipped`), a file lands in the directory and it is red, everything back is
`0 ran, 1 skipped`.

```
x3 gate -only "inline examples"        # template.json edited
== inline examples (case) RED          # the old engine: skipped, cached
```

⛔ **An imported package is weighed by what the importer can reach, not by its
whole text.** Measured on a pilot (v0.319.0): `return x` → `return (x)` in one
core function re-ran every importer, the core region 189 s with case 185 s,
though most of them never reached that function. The key now reads each imported
package as declarations. The seeds are every `q.Name` the importing package
writes — in its code, its tests and its example lines. A reached declaration
reaches the names it uses, and a reached **type** reaches **every exported method**
it has. That is why an interface call needs no implementor search: a value's
dynamic type is born in code that names it, so whatever calls its exported method —
an interface, reflection, `fmt` asking for `String`, `encoding/json` asking for
`MarshalJSON` — calls a method already in the key. An **unexported** method is
reached only when reached code of its own package writes `.name` (a selector, a
method value, a call through a package-local interface, a call promoted through an
embedded field): no other package's interface can name it and reflection does not
see it. Every
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
always reached, a reached type takes its exported methods and the unexported ones
reached code names. What lies outside is weighed **per example** too: the base is
every imported package's `var` and `init` closure, and on top of it only what this
example's reached declarations (test and fixture files included) and its own
`q.Name` pairs reach. An example whose key matches its last green is recalled; the
report lists it under `recalled`.

```
p.hidden() changes, nobody writes .hidden   -> one example runs as a witness, four recalled
p.local() changes, called via a local iface -> Local runs and its red is seen
p.promo() changes, called as Outer{}.promo() -> Promo runs and its red is seen
T.String / T.MarshalJSON changes            -> every example building T runs (fmt, json)
lib.A changes, only UA calls it             -> UA runs, UB recalled
```

Experiment `case reached type takes only named methods control experiment`: the
`case-method` tree, one limb per row; the previous engine says `5 of 5` and `2 of
2` on the two narrowing rows and turns them red, the four safety rows are red on
the change under both.

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

**The witness is spared only when nothing the build reads can have moved.**
When every example of a package is recalled, one still runs as the witness that
the package builds, with vet - unless the package's own lines and every
dependency as the importer's build sees it are unchanged since its last green.
A dependency that runs examples in this run is weighed by its declarations'
shapes (its own run, or its cached green, proves its bodies compile); an open
function (a `...` parameter: vet printf derives wrapper facts from the body; or
a type parameter: the body is instantiated in the importer) is weighed with its
body; a dependency with no examples, a whole plan (linkname, assembly, cgo, dot
import) or an interface method with `...` (loose) is weighed with all its lines;
`GOFLAGS` (as the child `go test` sees it) carrying any `-vet` turns sparing off,
because `-vet=all` opens body-derived facts (W815: `func Die() { panic("x") }`
becoming `{}` breaks the importer's lostcancel vet). A tagged or narrowed run
does not spare. A spared package is neither compiled nor run; its records are
written. Measured (pilot, one line added to a body in a core package): 92
packages ran, 66 of them only for the witness.

```
lib.Double body changes (lib runs its example)    -> use spared, not compiled, green
bare.Word body breaks, bare has no examples       -> use's witness runs, build red is seen
lib.Note(format, args ...any) starts fmt.Sprintf  -> use's witness runs, vet red is seen
logx.Reg body assigns a wrapper (loose package)   -> use's witness runs, vet red is seen
-vet=all, lib.Die stops panicking                 -> use's witness runs, lostcancel red is seen
```

Experiment (control): the arms in internal/cases/witness/witness.go; with the
`-vet` guard, the open-node weighing and the loose/proven guards switched off,
the printf, variadic, iface, ifaceonly and vet arms turn red (5 findings); with
only the "no examples" guard off the `quiet` arm turns red.

What else the build reads is weighed in every key (W811, W816; each was a false
green measured by an independent review, W810 / W815):

- **Files Go builds that are not Go** (`.s .S .c .h .cc .cpp .m .f .syso`
  ...) are lines of their directory, seen by the package and by its importers.
- **The toolchain the run uses:** every field of `go env -json`, asked once per
  environment with **the environment the child `go test` gets** (the machine
  share's `X3_CPU_GOFLAGS`, not x3's own `GOFLAGS`), outside any module, so
  `go env -w` counts. No allow list: a variable Go adds (`GOFIPS140` was
  missing from the old list) is weighed by itself. Only what was measured to
  move between runs is dropped: `-p=N` in `GOFLAGS` (parallelism) and the
  temporary `-ffile-prefix-map=<go-build dir>=/tmp/go-build` in `GOGCCFLAGS`
  (new on every call). `GOCACHE GOMODCACHE GOTMPDIR GOENV` did not move and
  stay: changing one drops the cache once.
- **Module sources outside go.mod's text:** a `replace` to a local path, the
  `use` and `replace` dirs of the effective go.work (with go.work and
  go.work.sum), and `vendor/`. Each one's go.mod/go.sum counts; its files count
  too unless the walk already reads it (a dir inside the measured tree whose
  module path is `<module>/<dir>`).
- **An interface method with `...`** makes its package seen whole by importers
  in the package digest (since Go 1.26 vet's printf calls the method a wrapper
  when ANY body of the package assigns a wrapper type to the interface).
- **A package whose own plan is whole** (`//go:linkname`, a bodyless function,
  `import "C"`, a dot import) weighs every package it imports with all their
  lines: what it reaches cannot be read from Go names. `//go:linkname hidden
  m/lib.hidden` names nothing in lib, and a change of lib.hidden's body kept
  the package key.

```
lib.Double body changes (lib runs its example)        -> use spared (not compiled), green
lib.Note(format, args ...any) starts fmt.Sprintf      -> use's witness runs, vet red is seen
lib.Name() string becomes int                         -> use's witness runs, build red is seen
type error in plain (no examples)                     -> use's witness runs, build red is seen
plain/nop.s breaks (+ lib.Name body, or alone)        -> use's witness runs, asm red is seen
GOFLAGS=-tags=foo opens a broken plain/tag.go         -> every example runs again, red is seen
X3_CPU_GOFLAGS=-tags=foo alone opens lib/foo.go       -> every example runs again, red is seen
GOFIPS140=v1.0.0 opens a broken //go:build fips140v1.0 -> every example runs again, red is seen
-vet=all, lib.Die stops panicking                     -> use's witness runs, lostcancel red is seen
lib.hidden body changes, use pulls it by linkname     -> use runs, its example is red
replace e => ./ext, ext.E changes its result          -> use's witness runs, build red is seen
plain/note.txt (embedded) removed / alias target      -> use's witness runs, red is seen
```

Arms: `internal/cases/testdata/spare` (bases `base`, `vetbase`, `lnbase`,
`envtag` and their variants), the cache-backed examples beside `Check`, and the
`toolchain`/`settled`/`builds`/`localTargets` examples in
internal/cases/toolchain.go. Control: the W816 arms `vet`, `envtag` (GOFIPS140,
X3_CPU_GOFLAGS) and `linkname` were run against the engine before W816 and each
was false green (0 findings); the W811 arms (asm, asmonly, tags, iface,
ifaceonly, replace) were false green before W811.

**A body-only change runs only the examples that executed that body.** Static
reach stops at the constructor: an example that builds an `App` reaches every
handler it registers. So every untagged run also records, per example, the
declarations it actually **executed** (`<package>#ran` in the case cache), and a
later run recalls an example whose key moved when the key's bodyless form
(signatures, directives, types, constants, variables, `init`) is unchanged and
none of the declarations it executed changed. The record comes from x3's own
marks, not `-cover`: each reached source goes into the overlay with
`x3hit(<n>);` right after every function body's `{` on the same line (lines and
compiler messages keep their numbers), plus one generated file per directory that
leaves its bits on `flag.CommandLine`; the generated test resets them before each
example and reads them after its cleanup. What runs before the first example
(package variables, `init`) counts for every example. Marked files are stored by
content, so the build cache compiles each one once.

```
cold() changes, Over executes it, Big only reaches it -> Over runs and is red, Big recalled
square.area changes, called through an interface       -> Area runs and is red
inc changes, called inside a goroutine                 -> Async runs and is red
cold(n) -> cold(m) (a signature)                       -> static rule: Big and Over run
an example leaves a goroutine running                  -> its record = its marks + every later mark (tail)
Kick wakes the goroutine Spawn left, late() changes    -> Kick runs (its record names late) and the crash is red
start() changes, run while the package loads           -> every example runs
an example with no record                              -> static rule
pad() gains a line, Lift sits below it                  -> Two runs (pad changed), Lift recalled
here() calls runtime.Caller, its line moved             -> Where runs and is red (out[0] = 17, want 16)
```

Declarations are weighed **without their line numbers** (the body's tokens,
directives included): a line added to one body shifts every declaration below
it, and a line-bearing weight reran every example that executed one of them
(pilot v0.379: 1 481 of 3 651 examples in one package after three added lines).
The line-bearing weight is kept for an example whose executed set holds a body
that reads source lines at run time (`runtime.Caller`, `Callers`,
`CallersFrames`, `FuncForPC`, `runtime.Stack`, `debug.Stack`, `debug.PrintStack`,
`log.Lshortfile`, `log.Llongfile`, any `.FileLine`, `AddSource`; aliased imports
resolve) and for every example of a package whose examples or test files name
one: `runtime.Caller(1)` reports its caller's line, so the whole set is weighed
by line, not only the reading body. Example records are named by file, owner and
the example's text, not by `file:line`, so an example below the edit keeps its
record. Experiment: `testdata/ran/lines` (with the line-bearing weights restored
the shift arm turns red: `len(out0.Recalled) == 1 does not hold`).

The record is added to static reach, never instead of it: an example whose key
is unchanged still runs when a body it executed changed (a goroutine another
example left can run code during it). An example that leaves a goroutine
running keeps its own marks plus every mark seen after it, written by a
generated `TestMain` at the end of the binary (only when the package declares
none): it over-counts, never under-counts.

No record is kept (the example falls back to the static rule) when an example
leaves a goroutine running and the package has its own `TestMain`, executes a body that may start a subprocess, a marked
directory's bits never reach the test, a file cannot be marked, or the run
carries tags. A cgo file's or a `//go:nosplit` body counts as executed by every
example. Experiment `run-time coverage control experiment green`: the trees under
`internal/cases/testdata/ran` differ in one line each; the cache-backed arms beside
`Check` assert each row, and with the marks switched off the `Over`/`Big` rows
turn red (`len(out0.Recalled) == 5 does not hold`).

**A dependency that only grew does not rerun the examples that never entered
it.** When an example's bodyless key moved but its own package's part did not,
the record also names each imported directory's *frame* (`<dir>#frame` in the
case cache: every declaration's bodyless digest, the package head, the non-Go
sources). The example is still recalled when every directory whose frame changed
only gained new functions, variables without a load-time initializer, types, or
new methods whose name no interface in the package's closure declares, which is
not a standard-library method checked at run time (`String`, `Error`,
`MarshalJSON`, `Scan`, `ServeHTTP`, ...) and which no other method or unexported
selector in that directory already spells, or only changed non-Go files that it
embeds into unexported variables. A new declaration named like a predeclared
identifier (`max`, `len`, `error`) is never one of those: unchanged bodies that
spelled the builtin now mean the new one. Bodies the example ran in that
directory, load-time ones included, allow the recall unchanged: an unchanged body
sees a new declaration only through a changed body it calls, a shape it reaches,
a shadowed builtin, an interface or a changed embedded file. They hold it back
only if a non-Go file changed and one of them names the variable embedding it,
or a declaration there changed shape and the directory imports `reflect` (a
reflecting body reads a type without naming it). A declaration whose **shape** changed (a struct
gains a field, a signature or a variable's initializer changes) holds the example
back only if a body the example ran reaches it **by symbol**: from every body in
its record (its own package's included, load-time bodies included) the engine
follows names through type, variable and constant declarations (field types,
embeddings, aliases, initializers) and never into a function that did not run. A
field added to a struct the example never reaches cannot change its result; a
positional literal names the struct, so its example runs and does not build.
Anything else stays on the static rule: a removed declaration, a new constant,
`init` or variable whose initializer runs code while the binary loads, a new
method named by an interface, a changed package head, an exported embedded file,
a missing frame record, or a shape change that comes together with any of those.

**A file's imports bind only that file; the package head is the closure.** An
import is file-scoped, so a file's import lines enter no other declaration: each
declaration's digests fold in the bindings it spells (`strings=strings` for a
body that says `strings.Cut`; a function's bodyless digest only those its
signature spells). Adding an import, or a whole new file, changes no existing
digest; moving an import a body spells to another path changes that body's
digest even though its text is the same. Dot imports, imports named like a
builtin and unnamed imports of a dotted module path enter every declaration of
their file. The package head is the package name, the transitive closure of the
package's import paths (module directories from their plans, the standard
library from the toolchain's `GOROOT`) and the file-level directives: an import
already in the closure links nothing new and moves no `init`, one outside it
does both. The package's own examples weigh the same head instead of every
file's import lines, plus the method names of every type they reach.

```
site gains clone.go (reflect, sync: already in the closure), a new
type, unexported function and variables, and site.go gains "sync"   -> 3 recalled, 1 witness
app (the examples' own package) gains extra.go the same way          -> 3 recalled, 1 witness
site gains func max(a, b int) int; Lift's unchanged body says max    -> Lift runs red
site.go's alt "m/one" becomes alt "m/two"; Who's body is unchanged   -> Who runs red
site gains import _ "crypto/sha3" (outside the closure)              -> all run, Routes red
```

Control: v0.381.0 (pilot) reran every importer of a package that gained one
file: `3458 shape changed inside the package, static rule` in the package and
`shape changed in lib/site (via lib/platform), static rule: its imports
changed` in each importer. On the `newfile` and `own` arms the same engine ran
4 of 4 (`4 shape changed in site, static rule: its imports changed`, `4 shape
changed inside the package, static rule`). Arms: `internal/cases/testdata/ran/heads`.

```
site gains Fresh() and Page.Slug(); site runs kinds() at load,
only Lift ran a body in site during its example                 -> 4 recalled, 1 witness
site.Lift's signature changes                                   -> Lift runs red, 4 recalled
site.Label (read while web loads) changes                       -> Banner runs red, 4 recalled
site.Code gains Tag(), the importer asserts its own tagger      -> all 5 run, Tagged red
site's unexported embedded note.txt changes, no body read it    -> 4 recalled, 1 witness
```

The symbol-level arms (`internal/cases/testdata/ran/shape`, App-shaped, so the
package's static walk reaches the struct through `web.Wrap`):

```
site.Shell gains a field, only Wrap's body names it             -> Wrap runs, 2 recalled
the same field, web.Wrap builds Shell with a positional literal -> Wrap does not build
```

```
RERUN app: 1 example(s) rerun, 2 recalled - 1 shape changed in site, static rule: the shape of a declaration changed and a body the example ran reaches it by symbol
```

Control: v0.359.0 ran 3 of 3 on the `field` arm (`3 shape changed in site, static
rule: the shape of a declaration changed`).

Arms: `internal/cases/testdata/ran/deps` (base, added, sig, initvar, iface,
body, embedfile) and `ran/shape` (base, field, posbase, posfield), measured by the
package examples of `internal/cases/ran` (they moved out of `internal/cases` so
that package stays under its time ceiling); `grown`, `reshaped`, `exposes`,
`changedDirs` and `loaded` carry their own examples in internal/cases/ran.go. Control: with
v0.353.0 the `added` tree ran 5 of 5 (`5 reach a changed declaration outside the
package`); counting the load-time body as the example's own also ran 5 of 5; until
v0.381.0 Lift ran on `added` and `embedfile` (`shape changed in site and the
example ran a body there`). The old `reflect` clause never fired: it asked the
module-internal import set, which never holds a standard-library path. Known limit: a method
called by name through reflection or a template (`{{.Slug}}`) is not seen; a new
method only matters there if text that already named it starts to resolve.

**Working copies that share one cache keep their own records.** The cache
directory is often shared by several checkouts of the same repository (a main
tree and its worktrees). The coverage record and the example keys are stored
under the same name in every copy, so the newest slot used to win, even when
another copy with different code wrote it. Each record now carries a seal of
the package's own files, and the coverage record also the frame of every
directory it reached; a run picks the slot that matches its own tree best and
the newest only on a tie. Measured on the pilot: after a worktree run, a body
change in one dependency reran 1016 of 1016 examples in the importing package
(`977 shape changed inside the package`); with the fix only the examples that
ran the changed body rerun. Fixture: `internal/cases/testdata/ran/trees`
(base, other, apart, body), measured by two examples in `internal/cases/ran`
that run `other` (different embedded file) or `apart` (different package code)
over the warm `base` cache before `body`:

```
RERUN app: 2 example(s) rerun - 1 ran a changed body in site; 1 shape changed in site, static rule: a body the example ran there names note, which embeds a changed file
RERUN app: 1 example(s) rerun, 1 recalled - 1 ran a changed body in site
```

Control: the first line is v0.382.0 on `body` after `other`; the second is this
release. With the choice reverted both examples fail (`len(out0.Recalled) == 1`).

**Every rerun package says why.** `x3 case` prints one line on stderr for each
package it runs, and the summary counts what came from the cache:

```
RERUN app: 1 example(s) rerun, 4 recalled - 1 ran a changed body in site
RERUN app: 5 example(s) rerun - 4 shape changed in site, static rule: the shape of a declaration changed; 1 ran a changed body in site
x3 case: 5 example(s) in 1 package(s) - 4 passed, 4 example(s) recalled from their record, 1 finding(s)
```

The same lines are in the report (`rerun:`, with `recalled:` per package). A dependency reached through
another package is named with its first hop: `shape changed in lib/render (via
lib/web)`. The `body` and `sig` arms assert the two different reasons.

**The gate carries those lines, green steps included.** A step's output is
dropped when it passes, so a gate step that runs `x3 case` used to hide which
package ran and why. Now any line that starts with `RERUN ` in the output of a
trial that ran this time is printed under that trial. The screen shows the real
reruns first, the line that ran the most examples first (`RERUN <subject>: <N> ...`,
ties keep the tool's order), at most 8 of them plus `... and M more rerun line(s)`.
A line whose reason starts with `routine: ` is not a change but the tool's own
habit (in `x3 case`, the one example that runs as the witness that the package
still builds); such lines fold into one line per reason at the end:
`... 80 routine rerun line(s) - routine: 1 runs as the witness that the package
still builds`. The full list, in the tool's order, is in the gate report (`rerun:`
on the step). The gate does not know which tool printed the line: any step whose
own report says what it ran again can use the same word and the same `routine: `
mark. A trial or step recalled from the cache carries nothing, since its output
tells yesterday's run.

```
== inline examples    (301615 ms)
  ran                                    exit=0 (want 0)
    RERUN cmd/admin: 16 example(s) rerun, 180 recalled - 16 cannot be narrowed (it reaches startProcess, whose body may start a subprocess)
    RERUN app/boot: 6 example(s) rerun, 2 recalled - 6 ran a changed body in lib/site
    ... 80 routine rerun line(s) - routine: 1 runs as the witness that the package still builds
```

Same step, nothing changed since: `== inline examples  (skipped: cached: nothing
this step reads has changed)` and no `RERUN` line.

Examples: `carry` in internal/gate/finding.go (a carried line, a recalled trial
carries none, an indented `RERUN` is not the word, ten lines shown as eight plus
`and 2 more`, a mixed input whose first shown line is the body reason and whose
two witness lines fold into one, `rerunN` reading the count) and `RerunLine` in internal/cases/cases.go. Control: with `carry`
returning the trial line untouched, the first and last examples are red; with
the sort and the fold removed, the mixed example is red.

**A change outside a body runs only the examples it can reach.** Only a package
variable whose initializer may run code while the binary loads (a call or
conversion, an index, a type assertion, a func literal, a channel receive, a
division or shift, a pointer dereference, `==`/`!=`, `any`/`interface`, a selector
on anything but an import name) counts as reached by every example. A variable
with no initializer, or one built only from literals, composite literals and
import-qualified names, is reached by name like any other declaration. A constant
named only inside the package's own function bodies keeps just its name in the
bodyless key; its value joins the run-time digest of every body that names it.
A constant named in a signature, a type, a variable, another constant, a test or
fixture file, or the example's own text is weighed whole, as before.

```
func spare() / var unset int / var label = []string{"a"} added -> all recalled (one witness)
var _ = reset() added, reset() writes factor                -> every example runs, Scaled red
func (b box) String() added (box now a fmt.Stringer)        -> Show runs and is red
const step 1 -> 2, Next executes it, Low only reaches it    -> Next runs and is red, Low recalled
var factor 1 -> 2 (no loading initializer)                  -> Scaled runs and is red, siblings recalled
```

Control arms: the trees under
`internal/cases/testdata/decl` differ from `base` in one place each and the arms
beside `Check` assert each row; with every variable treated as non-loading the
`loaded` row turns red (`len(out0.Recalled) == 0 does not hold`), and so do the
loading-variable rows of `testdata/ran/boot` and `testdata/reach/var`. Pilot
(one 1 775-example package, W581): a new `var x int` ran 1 775 of 1 775 examples
before; measured numbers are in the box `case-runtime-coverage-selection`.

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
under other tags) and `routine: 1 runs as the witness that the package still builds`.
Each package that ran also says where its seconds went, so a slow change is
read without a profiler:

```
x3 case: why apps/core: prepared 0.3 s, took 6.4 s: toolchain 6.1 s, binary 0.4 s, 3 example(s), 639 marked file(s)
```

`prepared` is marking and staging, `toolchain` is compile, link and vet, and
`binary` is the test binary's own run as `go test -json` reports it. Planning
walks each example's reach into the packages it imports once per distinct
seed set (VT pilot, one body edit in a shared package: planning 17.8 -> 9.3 s,
the whole direct run 32.2 -> 16.6 s, the same 20 packages and examples run).
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

The same holds for a red whose output carries an operating system's
*out of resources* sentence: Windows' `lacked sufficient buffer space` and
`only one usage of each socket address`, POSIX's `no buffer space available`,
`too many open files` and `cannot assign requested address`. None is a verdict
on the code; it is the machine under load. Measured on the pilot: a cold run
beside other gates lost one example to `dial tcp 127.0.0.1:5432: bind: ...
lacked sufficient buffer space`, the red was cached, and the next three runs
with nothing changed printed it again from the record. An example that falls
this way is `machine_exhausted` instead of `example_failed`:

```
$ x3 case -only exhausted
exhausted/dial.go:11 (Dial): machine_exhausted
	this run cannot measure: the machine ran out of a resource, so this verdict is the machine's - run it again: ... lacked sufficient buffer space ...
exit 1
```

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

<!-- x3-dist version=v0.384.0 capabilities=dc95d117c2c39a9cccc65afb8fbcf60ed2773fe49a845c5249fc246c6204eaf0 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
