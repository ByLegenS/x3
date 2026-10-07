# Control experiments, and the documentation gate

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 rename`

A Go name lives in two places: the code, and the `//x3:case` lines that call it.
`gofmt -r` and `gopls rename` reach only the first - an example is a comment - so a
package whose tests became examples keeps the old name in hundreds of example
lines after the code is renamed. `x3 rename` renames one **package-level** name - or a
field or method of a package-level type (below) - in
both, in one step:

```text
x3 rename [-dir <package>] [-dry] <old>|<Type>.<old> <new>

$ x3 rename -dir internal/orders -dry newRunner newOrderRunner
  handler.go:42:17 example
  ...
x3 rename: newRunner -> newOrderRunner in .../orders: 37 site(s) in 6 file(s) - 1 in code, 36 in examples
x3 rename: the renamed package type-checks; dry run, nothing written
```

**The object changes, not the word.** In the code, go/types names every use of
the same object; a local that shadows the name is a different object and stays.
An example line is parsed as Go (the setup as statements, every argument,
expectation and proposition as an expression) and only a name the example does
**not** resolve itself - the package's name - changes. The same word inside a
string, a field after a dot and a composite-literal key are left alone. Prose in
doc comments is not renamed, as with gopls.

**The package is loaded the way the example runner builds it:** its tests and its
own `x3fixture` files. The fixture tag is never given to the load - it would
spread to every dependency and close an import cycle; the fixtures are added to
the type check of this one package only.

**Nothing is written unless all of it holds** (exit 2, `REFUSED`): the new name is
already declared in the package; a local named `<new>` would capture a renamed use,
in the code or inside an example; the new name is one the example runner binds
itself (`t`, `out0`..., `x3...`) or a predeclared name; the old name is exported
(other packages may name it) or not package-level (a member is written `<Type>.<old>`); a file outside this build names
it. The renamed package is then **type-checked in memory**; a red check writes
nothing and exits 1. `-dry` prints every
`file:line:col` and whether it is `code` or `example`.

**Only the names change in a file that was not gofmt-clean.** A gofmt-clean file
is gofmt'ed after the rename, since a longer or shorter name shifts aligned
columns. A file gofmt would change on its own is written with the renamed tokens
and every other byte as it was, and the run says so:

```
x3 rename: kept protocol.go as written: it is not gofmt-clean, so only the names change (alignment not redone)
```

Measured in a real project: a doc comment with an example line right under its
text (no blank `//` between them) is one gofmt rewrites - it inserts the blank
line before each directive. Renaming 19 locals in one package re-laid three such
files, 12 comment lines were added and a frozen line count went red; with this
rule the same rename gives the six files byte for byte what renaming them by
hand gave. Control experiment: the `kept` arms of `rename control experiment`
(tree `rename-write:kept`) and the examples on `tidy` in
`internal/rename/rename.go`; with the rule reverted both go red.

Control experiment: `rename control experiment` in `x3.yaml` - counts, the string
and shadow arms, three refusals and a written tree.

### A field or a method: `<Type>.<old> <new>`

```text
$ x3 rename -dir internal/rename/testdata/member -dry counter.hits visits
x3 rename: counter.hits -> visits in memberfix: 13 site(s) in 1 file(s) - 6 in code, 3 in examples, 4 in comments
```

A member of a package-level type is renamed in the code (every identifier go/types
binds to that field or method: declarations, selectors, composite-literal keys,
uses promoted through an embedding type), in the example lines, and in the
comments that name it **qualified**: `counter.hits`, `(*counter).bump`,
`[counter.hits]`, and the first word of the member's own doc comment. The same
word in plain prose (`w.hits`, *"hits"*) is not renamed - it may be the member
or an ordinary word, and only the qualified form says which.

An example line has no types, so a member is found there by **position**: the
right side of a selector, or a key of a literal typed `counter{...}` (a literal
of another type is left alone). A selector is refused when another type of the
package declares a member of the same name - `c.total` cannot be told from
`o.total` without types. Further refusals (exit 2, nothing written): the new
name is already a field or method of the type; the member is exported, embedded,
promoted from elsewhere, or belongs to an interface (its implementations would
have to follow); and **capture** - a type embedding `counter` that declares its
own `count` would take `w.hits` once it reads `w.count`. That rename type-checks,
so after the in-memory check the uses bound to the renamed member are counted
against the code sites rewritten; a shortfall refuses.

Control experiment: `rename member control experiment` in `x3.yaml` on
`internal/rename/testdata/member` - a field and a method renamed with counts, and
five refusals. Breaking the capture count or the rival-selector refusal turns two
trials red.

### Any object, examples included: `-at` and `-all`

```text
x3 rename [-dir <package>[/...]] [-dry] -at <file>:<line> <old> <new>
x3 rename [-dir <package>[/...]] [-dry] -all <old>=<new> [<old>=<new> ...]
x3 rename [-dir <package>[/...]] [-dry] -map <file>      # "<old> <new>" lines; implies -all

$ x3 rename -dir internal/queue -dry -at queue.go:217 started ran
x3 rename: started -> ran (example local at queue.go:217): 3 site(s) - 0 in code, 3 in examples
  queue.go:217:19 example
  ...
x3 rename: example/internal/queue and 107 of its 107 example(s) type-check after the rename (0 did not compile before it)
```

`-at` renames the **one object** named `<old>` on that line - a local, a
parameter, a name an example declares in its `given=`, a field, a method, a
function, a type, a constant, a label. `-all` renames **every object of the
package spelled `<old>`**, each one checked on its own: a refused object is
listed with its reason and the others go ahead. `/...` measures every package under
the directory (in parallel, four at a time); nothing is written unless every
package is clean.

**The examples are type-checked, not parsed.** Each file's `//x3:case` lines are
turned into Go the way the example runner writes them - the setup, the receiver,
the call, every expectation and proposition, the file's imports, its
`//x3:import:` lines and the `case.imports` pool of `x3.yaml` - and checked
**together with the package**. Every identifier inside an example is then bound by
go/types, and each one is mapped back to its byte in the comment. So a name an
example declares and uses in a closure, a selector, a literal key, an outer name
of the same spelling, a string: each is told apart by the object it binds to,
never by its text. One example is one `//x3:case` line; a name declared in one
line is not seen by the next.

**Proof before writing** (exit 1, nothing written): the renamed package is
type-checked again in memory, and so is **every example that compiled before** -
an example that compiled and no longer does is red. Then every renamed identifier
must still bind to the declaration it bound to, and no other identifier may bind
to it. An example that did not compile before the rename is counted and listed
(`not proven, did not compile before`), not blamed on the rename.

**Refused** (the object is listed with `REFUSED`; with `-at`, or when nothing is
left to rename, exit 2): `<new>` is already declared in the same scope, in the
same example, or as an import of the package; a use would be **captured** by a
`<new>` declared closer to it; renamed, the object would **hide** an outer `<new>`
used after it - an import, a type, a package-level name (an object's scope starts
after its declaration, so `q, queueSetup := queueSetup(t)` is not hiding); `<new>`
is predeclared, a keyword, `_`, or a name the example runner binds (`t`,
`out0`...); the object is exported (or an exported field of a local struct), an
embedded field, an import name, `init` or `main`; with `-at`, a method that shares
its name with another method or interface method of the package; a name `-all`
lists as both old and new (a chain or a swap is two runs).

**What it does not see**, and what it does instead:

| Read at run time | Behaviour |
|---|---|
| a string spelling the name (`reflect` `FieldByName("hits")`, a template) | for a member or a package-level name: `WARNING ... a string literal spells hits at file:line` - the string is not changed |
| an exported name read by an encoder or a template | not reachable: exported names are refused |
| a method a type assertion expects at run time | renaming one of two same-named methods is refused under `-at`; `-all` renames both |
| a name in prose comments, `//go:linkname`, a build-tagged file of another platform | not renamed; a file outside the build that names a member or package-level name refuses |

Control experiment: the object arms of `rename control experiment` in `x3.yaml` on
`internal/rename/testdata/local` - an example local in a closure, a code local
beside a constant and a field of the same spelling, `-all` over the three, the
warning for a `reflect` lookup, five refusals (hiding an import, a name the
example already declares, hiding a type, a package-level name, an import name),
a conversion that only the type check of the renamed package catches (exit 1)
and a written tree. Each refusal was switched off once in the source: its arm
turned red (the hide and same-scope arms then exit 1 on the type check, the
import arm is refused as a capture, the exported arm renames, the conversion
arm passes with the type check off).

**One load for `<root>/...`.** Every package under the root is read by a single
`go list` (tests, fixtures, `//x3:import` and `case.imports` packages included) and
type-checked from that one graph. Measured on a real production Go application's
core module (97 packages, `-dry -all`, same output line for line): 53 s and 45 s
loading each package on its own, 3 s and 2 s with the one load.

### Exported names across the workspace: `-wide`

```text
x3 rename -dir <package> [-dry] [-config <file>] -wide [-force-runtime] -at <file>:<line> <old> <new>
x3 rename -dir <package> [-dry] [-config <file>] -wide [-force-runtime] -all <old>=<new> ...

$ x3 rename -dir internal/rename/testdata/wide/lib -dry -wide -at lib.go:7 Amount Cents
  lib/lib.go:7:2 code
  app/app.go:25:36 example
  deep/deep.go:14:39 code
  ...
x3 rename: widefix/lib: 1 object(s) renamed, 0 refused; 8 site(s) in 3 file(s) - 6 in code, 2 in examples
x3 rename: 2 importing package(s) of the workspace and 1 of their 1 example(s) type-check after the rename (0 did not compile before it)
```

Without `-wide` an exported name is refused. With it, the `go.work` workspace (or
the module) is loaded once; every package that imports the target, directly or
through another package, is **type-checked from source** in dependency order, so a
package that never imports the target but selects its field through another
package's value (`app.Default.Amount`) is found too. Packages that take the target
only in their tests, fixtures, `//x3:import` lines or the `case.imports` pool are
checked with their examples as well. A use is tied to the declaration by its
position (file and byte), so a generic instantiation counts as its origin.
**Proof before writing** is the one of `-at`/`-all`, over every one of those
packages and examples: nothing is written unless all of them type-check and every
renamed use binds to the renamed declaration. One package per run: `-wide` with
`<root>/...` is refused.

**Refused** - names read at run time, which no type check sees:

| What | Behaviour |
|---|---|
| an exported field with no struct tag | refused: encoders write and read it under its Go name; a tagged field is renamed with a warning (gob reads the Go name whatever the tag) |
| a member a template (`{{ .Title }}` in a Go string or a non-Go file of the package directory) or a string equal to it (reflect) reads | refused; `-force-runtime` renames it and lists the strings that do not follow |
| an exported method another package's interface declares (`String`, `Close` ...) | refused: a type assertion may ask for it at run time |
| a name `//go:linkname` reaches anywhere in the workspace | refused |
| a name an external test package (`package x_test`) names | refused: that package is not type-checked here |
| an importer that does not type-check from source before the rename | the whole run is refused |

Packages outside the workspace cannot be seen; a module others import from outside
is not a candidate for `-wide`.

Control experiment: the `-wide` arms of `rename control experiment` on
`internal/rename/testdata/wide` (a target, an importer with an example and a
template, a package that selects the field without importing it, a linkname, an
external test): two renames with counts, six refusals, `-force-runtime`, a
conversion only the importer's type check catches (exit 1), and `/...` refused.
Each refusal and the importer check were switched off once in the source: the
arm turned red.

**The pool and a name the package declares.** The `case.imports` pool comes from
`x3.yaml` of the working directory, or from `-config <file>` (a file that cannot be
read is an error, exit 2, not an empty pool). The runner imports a pool package only
when an example names it, so a package that declares the pool package's name at
package scope (`type action int` beside a pool entry `.../action`) compiles and
runs its examples. The mirror leaves such a pool entry out the same way: Go forbids
one name in both the file and the package block, and writing the import anyway
refused a building importer with `action already declared through import of
package action`.

```
$ x3 rename -config internal/rename/testdata/widepool.yaml \
    -dir internal/rename/testdata/widepool/lib -dry -wide -at lib.go:6 Half Halve
x3 rename: 1 importing package(s) of the workspace and 1 of their 1 example(s) type-check after the rename (0 did not compile before it)
```

Control experiment: the `widepool` arms of `rename control experiment` - the
importer's example green under the same pool, the rename above green with `not:
already declared through import`, an unreadable `-config` exit 2. With the scope
filter removed from the mirror the rename arm is refused (exit 2) with `sort already
declared through import of package sort`; the binary before the fix gives the same.

## `x3 move`

`x3 rename`'s counterpart for a **path**. A directory renamed by hand leaves its old
path in configuration, documentation, import paths and package qualifiers, and a
text replacer does not know where a path ends: replacing `cmd/apidoc` also rewrites
`cmd/apidocs` and `services/cmd/apidoc`. `x3 move` moves one directory or file and
updates every reference to it in the same step:

```text
x3 move [-root <dir>] [-config <file>] [-dry] <old> <new>

$ x3 move -dry core/checkrun core/checkup
x3 move: core/checkrun -> core/checkup (git ls-files; the move is a git mv)
  apps/x/main.go: 1 path(s), 4 qualifier(s)
  core/checkrun/run.go: package clause
  x3/gate.yaml: 2 path(s)
x3 move: package checkrun -> checkup: 1 package clause(s), 4 qualifier(s)
x3 move: move.keep leaves alone: db/migrations/**, x3/work/archive/**
  kept: x3/work/archive/2026-09.jsonl: 3 mention(s) of core/checkrun, not rewritten
x3 move: 3 file(s) of 412 read change: 3 path mention(s), 4 qualifier(s), 1 package clause(s)
x3 move: dry run, nothing written
```

Both paths are written from the root (`-root`, default `.`). In a git work tree the
files read are the ones git tracks and the move is a `git mv`; outside one, every
file under the root is read and the move is a plain rename - the first line says
which. Binary files are never read.

**A path changes only on a path boundary.** Before the mention there must be no path
character (`./` is allowed); after it, a `/`, the end, or a character that cannot
continue a name (a full stop ending a sentence counts; `.md` does not). A
single-element path (`docs`) changes only where it is written as a path - followed by
`/` or preceded by `./` - so the same word in prose stays. The Windows spelling with
`\` is changed with the same boundary.

**A Go package moves with its name.** When the moved directory sits inside a Go
module under the root, its import path (and every import path below it) changes
like any other mention. When its last element is the package's name and the new last
element is a Go identifier, the package clause changes too (`p` and `p_test`), and
so does the qualifier in every file that imports the package without a name - in the
code and in the `//x3:case` lines, read with `x3 rename`'s own example reader, so a
local of the example or a word inside a string stays. A `main` package, a package not
named after its directory, or a new element that is not an identifier
(`live-probe`) keeps its name, and the run says why.

**What it does not touch is declared, and said.** `move.keep` lists globs a move
never rewrites - a published migration, an archive, a history ledger: there the old
path is that day's truth.

```yaml
move:
  keep:
    - db/migrations/**
    - x3/work/archive/**
```

With no `move.keep` the run says so on every run: every text file is rewritten, a
migration and a ledger included.

**A kept baseline file is not written either (2026-10-07).** After the move, `x3 move`
records it (`moved:`) in each baseline file whose records it reaches, so the next run
holds the moved debt. That writer did not ask `move.keep`: a gate baseline listed
there got a 14-line `moved:` record and had to be put back by hand. Now a baseline
file under `move.keep` is left byte for byte and named:

```text
  base/gate.yaml: listed under move.keep - not written; the move is not recorded over its 2 record(s), 0 of them on the moved path (take it out of move.keep to carry them)
  base/arch.yaml: move recorded over 1 record(s), 1 of them on the moved path
```

Its records are not carried; a record the move reaches may be born on the new path.
A project that wants the move carried there takes the file out of `move.keep`.
Measured by the `kept` arm of `move carries baseline records`: the kept file is
compared with a byte copy, the unkept one must differ; red on the engine before this
change.

**It proves itself from disk** (exit 1, `RED`): after writing, the files are listed
and read again, and a file outside `move.keep` that still names the old path is
named; the kept files that still name it are counted as declared. Every package
directory a Go file changed in is then loaded and type-checked the way the example
runner builds it; a red check names the errors. The move is already written by
then - undo it with git.

**Nothing is written** (exit 2, `REFUSED`) when the new path already exists (no
merge), the new path is inside the old one, the old one does not exist or git
tracks nothing under it, the new path leaves the Go module the old import path
belongs to, or the new package name would be captured in an importing file - a local
of that name, another import under that name, or a package-level declaration of
that name in the importing package; a predeclared name is refused too. `-dry` prints
every file with its counts and writes nothing.

**A path written from the file's own directory moves too.** A mention that is not
written from the root - `../../data/schema.json` in a Markdown link or a Go test, a
bare `page.html` beside the file that names it - is resolved against the directory
of the file it stands in; when it lands on the moved path it is rewritten from that
file's directory after the move, with its `./`, `..` and `\` spelling kept. A file
inside a moved directory that names something outside it with `../` is rewritten
from its new directory. Two spellings are never produced: a bare name that would
need a directory, and a `..` where the mention had none (`//go:embed` refuses `..`
and the type check does not see an embed pattern). Every mention of the moved name
that does not resolve - a URL `/page.html`, a bare name read from another directory
- is listed, never passed over:

```text
x3 move: 1 mention(s) of schema.json are not rewritten - look by hand:
  docs/notes.md:83: "schema.json" - not rewritten, look by hand
x3 move: 7 file(s) of 1779 read change: 6 path mention(s), 0 qualifier(s), 0 package clause(s), 2 relative mention(s)
```

A bare directory name with no `/` after it is a word, not a path, and is neither
rewritten nor listed.

**A Go file keeps its bytes.** The law of `x3 rename`: a gofmt-clean file is laid
out again after the edit (a longer path shifts an aligned column); a file that is
not gofmt-clean changes only where the path changes, and the run names it -
`kept config.go as written: it is not gofmt-clean, so only the paths change`.
Measured on a production repository: the old move ran gofmt over such a file and a
frozen line count grew from 1143 to 1147.

**A generated document is not edited by hand.** The documents the `emit` section
declares (their `out`, a `{field}` placeholder read as `*`) are never rewritten -
the next `x3 emit` would overwrite the edit, and `-check` would call it stale. When
one names the old path, or its template, a data file or a part is moved or
rewritten, the run says so: `docs/map.md is generated by emit document map and its
source tmpl/map.tmpl changes: it is not edited by hand - regenerate it with x3 emit
-only map`. After the move a generated document that still names the old path is
counted apart from the residue - stale until `x3 emit` writes it, not red.

Control experiment: `move control experiment` in `x3.yaml` - a dry run on a
git-tracked fixture run twice (the first must not have written), the boundary arm
(`cmd/apidocs` beside `cmd/apidoc`), counts, the proof from disk, the kept file read
back unchanged, a configuration without `move.keep`, a Go package moved with its
clause and its qualifiers in code and example then type-checked, and two refusals:
a clash and a capture. Breaking the boundary, the keep list or the qualifier rewrite
turns its arms red - the last one through the type check. Five arms measure the
rest: a caller that is not gofmt-clean is kept as written, a bare name and a
`../../` path are rewritten, a URL is listed with its line, a moved file's `../`
mention is rewritten from its new directory, and a generated document asks for
`x3 emit` instead of being edited. Each is red on the engine before them and red
again with its rule taken out.

**The baseline moves with the path.** A baseline record's identity is a digest of
its rule, its path and its content, so a moved file's held debt would be born on the
new path (and a gate record, which carries the file inside its content, with it).
After the residue check `x3 move` writes the move into every baseline file under
`baseline.dir` that holds a record the move can reach - one on the moved path, one
on a file the move rewrote (its content may carry the path), or one whose path is
not on disk (a gate record keyed by its step: its content cannot be read) - and
names each. A file whose records all sit on untouched paths is left unwritten and
counted (measured on a production repository: one move wrote `moved:` into 20
baseline files; with the reach rule, 3):

```
  base/arch.yaml: move recorded over 1 record(s), 1 of them on the moved path
  base/gate.yaml: move recorded over 2 record(s), 0 of them on the moved path
x3 move: 2 baseline file(s) carry the move; the next run holds the moved debt under its old record and -update-baseline writes it REKEYED
```

```yaml
moved:
  - from: cmd/panel
    to: cmd/console
```

The next run rebuilds an unheld finding on the old path (the path itself and every
boundary mention in its content, also in the digit-free form a position-proof
content uses) and takes over the record it finds; nothing is born and no record is
dead. `-update-baseline` writes it `REKEYED <old> -> <new>`, the count unchanged, and
drops `moved:` (a narrowed refresh keeps it). A frozen count keyed by the path follows
the key the move rewrote. A carried record with nothing on the new path to match
dies by name: `... x3 move carried it from cmd/panel to cmd/console and nothing there
matches it`. A plain `git mv` is the bound: the engine never sees that move and the
debt is born on the new path - move with `x3 move`.

Control experiment: `move carries baseline records` in `x3.yaml` - a driver gate
moves a package, measures arch, gate and freeze, refreshes and measures again (0
born, two `REKEYED`, no growth); the same move without the engine is `dead_baseline`;
the unmatched record is named. With the recording turned off the first arm is red
(the moved package's finding is born).

## Control experiments

**No gate here is trusted because it is green.** Every capability has a pair in
`check.ps1` — one tree it must pass and one it must fail — and the pair is the
evidence, because a green nobody has seen fail proves only that nothing ran. The
step names below are the ones the gate prints.

| Step | The pair, and what only the red half proves |
|---|---|
| `control experiment` | a well-formed sample `0`, a broken one `1` |
| `single git gateway control experiment` | one tree four ways: the gateway alone `0`, a second call site `1` **named by the rule**, the same call written in a **comment** `0`, and the same call in a **test that plants its own repository** `0` — the last two are what separate a rule from a text search |
| `absent git control experiment` | the **same tree and the same two commands** run twice, with `PATH` emptied the only difference: with git installed a tagless copy says `unreleased` and a non-repository says it is not a working tree; with git gone **both** say one sentence and neither says the other's — so the answer cannot have come from the tree |
| `case control experiment` | an example that holds, one whose value is wrong, one with no payload, and one **nothing ran** — the last is why `never_ran` exists; plus an example using its file's imports `0`, and a tree with one broken example `1` **whose three sound neighbours still passed**; a declared import no example names `1` in the tree that declares it and `0` from a narrower scope — **same tree, same setting, only the scope moves**; and a red example whose code writes its own log, so the finding must still speak the gate's sentence; a file that does not parse `1` called `does_not_parse` and the same tree readable `0`; and a proposition its predecessor rules out `1` **whose two neighbours still ran** |
| `case crash control experiment` | one tree of four examples, one of which kills the process: `1` with **exactly one** finding, `crashed`, naming the example it died in — and the **three beside it still ran**; then the same three with nothing to kill the run, `0` |
| `case cache control experiment` | one tree of two packages, one of which **imports** the other, asked thirteen ways: a cold run stores both and a second run measures **neither**; one file touched and the importing package - whose own file never changed - is measured again; the setting changed and **both** are measured again; `-no-cache` walks past a full box; a red example is measured again on every run because a red is **never stored**, and mending it is a miss too; one worker and eight produce the **same findings**; the report carries a time for every example; and an identical run opens **not one new work directory** |
| `case ceiling control experiment` | one 4s package under a 2s ceiling: `1`, every unmeasured example `over_ceiling` and the fast one still **passed**; the same tree asked **again** gives the **same** answer, not another innocent; the same tree under a 90s ceiling, `0` with three passed; and the crash fixture still `crashed` naming `Race` |
| `held verdict carries its address control experiment` | three entries asked by path: every judged entry names where it is held, no `free` entry names anything, and every address is either `<document>#<box>` or `<file>:<line>` |
| `case type declaration control experiment` | one shape asked twice: a fake the setup **cannot** write `1` (`does_not_build`), the same tree with the type declared `0` and **no file left behind**; then the declaration's own laws — a declaration no example names `1` (`dead_type`), one written under the `package` clause `1`, a method on a type the package already has `1`, and a declared name that collides `1` **blamed on the example that named it, its neighbour still passing** |
| `language gate` | the repository `0`; a planted word `1`; a green tree with its allow list `0` **and without it `1`** — an allow list never seen to change an answer is decoration |
| `docs gate` | this repository `0`; a rule whose counterpart directory cannot exist `1`; and on a planted repository whose single commit excuses one rule by name, that rule `0` while a second rule the reason does not name stays `1` |
| `roster control experiment` | one tree, seven readings, only the *question* changing: two configured checkers really called `0`; neither called `1`, the red naming **both** — a stale roster would have named neither. Then a script naming one of them **only in a help string**: the old reading `0` (the blindness), `strings: "exempt"` and `invocations` both `1`. Then a colon command genuinely called: the old reading `1` — a false red on a checker that runs — and `invocations` `0` |
| `configuration section registry` | a settings section the engine reads and the roster claims `0`; the roster gone stale `1`, naming the section. The green half on this repository's own tree is the `arch gate` step |
| `expectation scope control experiment` | an expectation naming a fixture tree `0` — the walk enters a skipped directory only because a rule declared it; the same tree with the expectation unnamed `1`, which is the blindness itself; the run started **inside** the named tree `0`; and the directive deleted `1` |
| `arch control experiment` | a green/red pair for every rule kind and every escape hatch: `absent` present, missing and dead; `skip` off, on and dead; `comments` read, exempt and embedded; `relativeTo` both ways; `minimum` met and short; `exclude` applying, dead and emptying the rule |
| `agent scope hook control experiment` | one planted repository, twenty-six calls: a path inside the scope passes in silence while the same call on `src/` is refused **naming the scope**; the same input on another branch passes; a listed agent is scoped on any branch and an unlisted one is not; an unlisted shell command, an unreadable command line, a broken input and a missing scope file are each refused, never passed; `$HOME/a.md` and `%USERPROFILE%/a.md` written from inside `docs/` are refused by their opened reading; an agent call with no scope file passes **saying so**; `git -C docs status` passes while `git -C src` and `git -C docs push` do not; a quoted here-document commit message passes while one with a second `$(` or an unquoted tag does not |
| `freeze control experiment` | the surface green, one name added red, and **`-update` on the grown tree red with the file unchanged** |
| `freeze count control experiment` | held, grown, shrunk and capped trees, each also under `-update`; the capped key stays out of the baseline **and the update itself exits `1`** |
| `surface control experiment` | six directions on one tree: **no baseline** (red, because growth is green here), recorded, untouched, a changed signature red **and naming its caller**, the same break allowed, a symbol *added* green, and the allow gone dead |
| `freeze scope control experiment` | one tree five ways; the exclusion written properly is `0` **on a tree built to be red without it**, and the same intent written as `"!…"` inside `sources` is `2` |
| `baseline control experiment` | no baseline, written, re-run, **the same debt moved down the file** (`0`), grown, `-update-baseline` refused, and one debt paid leaving `dead_baseline` |
| `baseline shrink control experiment` | one tree carrying **both** a paid debt and a new one: locked (`1`, `dead_baseline` named), then `-update-baseline` — the drop lands and the growth does not, both named, the file holding one entry; the re-run keeps the real finding (`1`) and has no `dead_baseline` left. Then a `count` that disagrees with its own list (`2`) |
| `secrets control experiment` | clean, leaky, exempted, a dead exemption, and this repository — the clean and exempted rows are what separate a gate from a noise generator |
| `secrets ignore control experiment` | a noisy tree with and without its exclusions, **then a real address added back**, then a dead exclusion, then a lookaround (`2`) |
| `comments control experiment` | inside the limit, one line over, exempted with a reason, an exemption that silences nothing |
| `boxes control experiment` | finished work left open, unfinished work closed, a condition unmet then met, two boxes closed with and without proof, and a `sql` criterion whose DSN is empty — counted, never green |
| `boxes document list control experiment` | the same for a document list, and **the same tree with one state declared `open` then `silent`** — one line of configuration decides, nothing else changes |
| `boxes criterion fidelity control experiment` | criteria that stopped measuring, each red with a control that removes the rule and returns the tree to green. Its fourth row is deliberately **green**: a test that was never written, measured by exit code alone, with no `output` |
| `boxes batch control experiment` | one tree measured twice, batched and not, three times over: a package whose criteria all hold, **a test that was never written beside two that pass**, and one criterion holding while another in the same package does not. The evidence is not the exit code but the **reports being identical** with and without batching, and the unmet criterion being the one the report names |
| `boxes unmeasured control experiment` | one tree of four closed boxes, run twice with one line of configuration between the runs: the box whose test skipped reads `box_unproven` without it and `box_unmeasured` with it — **still red** — while the box whose test failed and the box whose test was never written stay `box_unproven` either way, the box whose test passed says nothing either way, and both runs name the **same boxes**. A text written as the proof and as its absence is exit `2` |
| `arch produced path control experiment` | one tree run in **two machine states** — a clean checkout and one that has been built — against three settings. With no hatch the rule is red before the build and green after, which is the row that proves it measures at all; `absent` reverses that and is red on exactly one of the two machines; `skip` is green on both, while a pattern that sifts nothing is still red (`skip` dead, in the row above)
| `boxes measurement shell control experiment` | the same tree, the same version, **two shells**: without the variable a closed box is red (`1` finding), with it the run is green (`0`), and the neighbouring box - measured in both - is what proves the difference came from the shell and not from the tree. The report names the variable and whether it was set in each run, and `summary.blind` says `1 output`. A fourth row asks a variable **nobody declared**: a `sql` criterion names its own connection variable, so the shell line carries it and the cause reads `1 environment`. A declared variable with no name is `2` |
| `boxes dead selector control experiment` | one tree, four questions, and **not one criterion able to run**: the runner does not exist, so every closed box is `box_unmeasured` and the static answer is the only one there is. A selector no declaration matches is `box_dead_selector`; a selector whose name is declared **outside the place the criterion looks at** is the same code with the place it went to named; a selector still reached and an **open** box whose check is not written yet both say nothing. The same tree with the rule undeclared carries **zero** findings of this code - the red comes from the declaration, not the tree - and a tree in which no declaration can be read is `2` |
| `boxes connection fallback control experiment` | one criterion, one list of two variable names, two shells: with neither written the criterion is unmeasured and the cause is `environment`; with the **second** written the cause becomes `connection`, which is the proof the fallback name was really used; and the report names every place it looked |
| `boxes dead step selector control experiment` | one tree, one gate script, five questions and **no box involved**: a step selecting a name nothing declares is `dead_step_selector`; an alternation whose other half is gone is the same code naming the dead half, so a surviving alternative cannot hide it; a step naming a place the matching names sit outside of is told apart; a live selector and a subtest selector (`Name/case`) say nothing; every finding carries the gate line it came from, and the same tree with the rule undeclared carries zero |
| `boxes table row criterion control experiment` | one tree, four boxes, one question moving: a criterion written **inside a table row** is read and measured — red with its own name when it does not hold, silent when it does — a criterion **quoted** as prose is not invented into one (the box stays `box_uncovered`), an escaped pipe keeps the pattern whole, and the plain line form reads exactly as before |
| `boxes holds control experiment` | one tree asked nine ways, only the *question* moving: a file no criterion names `0` over a list that really carries criteria, a test only a **fragment selector** holds `1` while a search of the documents for its full name finds nothing, the same tree with no selector declared `0`; then a second tree where the criterion names a place — the same declaration outside that place `0`, inside it `1` — and the outside one `1` again once a gate script that calls it by name is declared; then the same name with **no file left in the tree**, asked three ways — with no place `1`, from a place the criterion never runs in `0`, from the place it does run in `1` |
| `boxes holds selector control experiment` | one tree, four questions, only the *declaration* and the *place* moving: a gate line handing its runner a **fragment** of the name holds the file `1` with `how: pattern`, the same name outside the place that line names `0`, the same tree with no reading declared `0` — the direction that proves the hold comes from the declaration and not from the tree — and a reading that reaches no line `2`. One row measures the fixture itself: the whole name is spelled on **zero** lines of that gate, so a word match could not have found it
| `boxes holds bindability control experiment` | one tree, one gate script, four questions: the gate names an exported package-level test `1`, carries an unexported name `0`, carries a method name `0`, and opens a file by its path `1`. Two rows measure the fixture itself — **both discarded words really are on a line of that gate** — so the green is a decision and not an empty search; two more read the report, where `names` and `bindable` show the narrowing |
| `holds comment control experiment` | one tree, one gate written three ways, six questions: a name only a **comment** carries holds nothing - a PowerShell `#`, a Python `#`, and a name written after a real step - while the same scripts' code lines still hold, and a `#` **heading** in a markdown procedure document is not a comment at all. One row measures the fixture: those three names really are spelled on three comment lines |
| `holds unwritten criterion control experiment` | one tree, five questions: an **open** box naming a file whose exam is not written yet holds nothing, the same writing whose exam **is** in the file holds, the same writing in a **closed** box still holds, and a criterion naming a **directory** holds the exam under it but not its namesake elsewhere. Two rows measure the fixture: the criterion really names the file, and the name really is absent from it |
| `boxes dead evidence control experiment` | one tree, one document, five questions: a **closed** box whose prose names a proof no declaration carries is `box_dead_evidence` at the prose line; the proof that is still there and an **open** box's unwritten one stay quiet; a name carried only by the criterion is measured once, as `box_unproven`, and never as evidence; the declared policy reaches the finding; and a reading that names nothing is `2` |
| `configuration relative path control experiment` | one configuration, two working directories, one answer. A run records a baseline from one directory `1`, and the file lands **beside the configuration** and beside neither working directory; the same configuration read from a second directory finds it, `0`; a new debt is `1` from both. Bound to the shell instead, the first run leaves the file in the wrong tree and the second sees no baseline at all |
| `syntax control experiment` | parsing, broken, a parser that is not installed (red), and the same check under `missing: "warn"` |
| `syntax ignore control experiment` | two trees and two settings, only the *question* moving: another language's own word red without the exclusion — the blindness itself — and green with it; the real violation red under both, so the exclusion is not hiding it; the same exclusion on the tree it excludes nothing in, `dead_ignore`; and an exclusion written on a parser check, `2` |
| `syntax subject control experiment` | one tree of three files, five questions: no condition at all reads all three — the blindness, a check reading files it is not about; `holds` reads one and counts two eliminated; `lacks` drops the file carrying the gate marker; a condition no file meets is `empty_scope` **naming the condition**, not a silent green; and a lookaround in it, `2` |
| `scope control experiment` | inside the lane, crossing it, crossing with a reason; then the branch form both ways, including a violation in the first commit under a clean one, and a closed lane |
| `test control experiment` | six directions on one tree, including **a full run when a file belongs to no unit** and a cache that answers, then measures again once the file changes |
| `record` / `replay control experiment` | a ledger whose credential header and planted key are hidden **while an ordinary field is still there**; then a replay without a `normalize` rule (red), with it (green), and against a drifted application (red) |
| `mutate control experiment` | one tree, nine directions: a well-tested package where every mutant is **caught**, a package whose test asserts nothing where every mutant **survives**, a package no test binary links (`no_test`, and **not one run launched**), an **embedded query** whose condition is caught while its `LIMIT` survives, the quick scope narrowed to the one changed file, a scope that produces **no mutant at all** (`empty_scope`, red — a gate that measured nothing would otherwise print a perfect score), a dead exclusion, survivors forgiven with a reason (green), and a forgiveness with nothing left to forgive (red); then the same scope **priced** — no launch, no green, and the real run's launch count lands between the two bounds — and the **worker ceiling** measured at half the processors |
| `outbound control experiment` | a call recorded through the proxy, the same answer served **with the far side shut down**, and an unrecorded call refused `502` |
| `guard control experiment` | green, blocked and warned — the launched command proves it ran by writing a file, and the blocked row proves it did not |
| `guard selection control experiment` | one file, only the flags changing; a mistyped tag exits `2` rather than skipping nothing |
| `multi-step trial control experiment` | the same trial green, red once an import is *written* into the copy, red on an empty removal — and **zero working areas left behind, the reds included** |
| `effective control experiment` | agreement, divergence under `block`, the same divergence under `warn` |
| `adoption section weight control experiment` | one tree, two configurations. A settings section and a directive-backed section under `block`, `0`; the same tree with one section declaring one rule, `1`. Then the report's own numbers, so the silence is measured and not accidental: the settings section **is named, has names inside it, and holds zero rules**; the directive section is weighed by the tree, not the configuration; a real rule list is still counted; and the red names the section and where its weight was read |
| `adoption policy control experiment` | one tree, eight settings. The first two are the measurement: the same tree, the same `block` default, `0` with the known code excepted and `1` without — the exception is the only thing that moved. Then the escapes: no reason, a bare word, an unknown code, a missing `"*"`, and `dead_policy` excepted from itself, each `2`; and an exception matching nothing, `1`. The last two rows read the report itself — the silenced finding is **still there, named, with its reason**, and the exception is counted against what it touched |
| `update control experiment` | installed, a planted checksum refused, the version gate both ways, and **a pin disagreeing with a release whose own checksum list is perfect** — which is exactly how a compromised release looks |
| `testdb control experiment` | a foreign name refused at the gate (`1`) **and our own name reaching an unreachable server (`2`)** — a gate that refused every name would also exit `1` |
| `expect control experiment` | the count met, one guard short, the directives deleted, and the same tree with no expectation |
| `public leak gate` | the published documents clean, and a planted tree in which **every** forbidden pattern speaks |
| `reference split control experiment` | this document really split: **no table lost and no marker left behind**, and the families it derives identical to the pages the settings cap; then a table under no page and a marker with no table, both refusing to publish |
| `public size gate` | every published document under its own cap, then **each cap in turn** asked with one line too many — one document over the line would leave the other caps unmeasured |
| `public language gate` | the published documents in one language, and the template's **real** maintainer note planted; the plant is not invented text, so the experiment measures the assumption too — a note rewritten in English would leave the gate unable to prove itself, and it says so |
| `comments baseline identity control experiment` | one file, four steps: the debt recorded (`1`, one entry written), the same tree frozen (`0`), **a new over-long block in that same file `1`** — named, with the old debt still held — and the frozen block re-indented `0`, because identity is the text and not its shape |
| `split configuration roster control experiment` | one tree asked twice, only the rulebook moving: a rule and a section declared in an `include` **part**, the rulebook naming neither `1` — both named in the red — and the rulebook naming both `0`. The same run reports that the part's rule really ran, which is the whole point: a name the engine is enforcing cannot be missing from the set of names in force |
| `published control experiment` | one planted world — a bare remote and a repository that knows it — asked as each step is taken: the tag never created `1`, the tag created and not pushed `1`, a lightweight tag pushed `0`, an annotated tag pushed `0`, the tag then moved to a later commit `1` (`tag_differs_on_remote`); then this repository's **own** publication `0`. The unpushed row is the release that was really made and could not be downloaded; the moved row is the annotated read staying red where it must |
| `dist gate` | the publication current, the same question asked with a deliberately wrong document hash, and a copy of the publication with one page missing |
| `leftover control experiment` | one program run five ways, only what it leaves moving: a measured command that forks a grandchild and returns is `1` with the **grandchild and its console host named**; the same program told to leave nothing is `0` and silent, **ten runs out of ten** — the row that proves the count is not noise; the red tree with `leaves` declared is `0` and still names them; the permission written with no reason is `2`; and the **same program started before the run** is `0`, uncounted and still alive afterwards, which is the false positive this gate would be useless for having |

Whatever cannot be arranged from a shell — a database, a network, a fake driver,
a mapping, a retry — is control-tested in Go instead, to the same rule: each
green is shown next to the red that proves it was measured.

## The documentation gate

This repository holds itself to the rule it ships: a change under `internal/` or
`cmd/` must carry a change under `docs/` in the same diff. The gate is
[`x3 docs`](docs.md#x3-docs) reading this repository's own `x3.yaml` — the same command
any project would run.

```
docs: none (code-changes-carry-documentation) - <why the reader loses nothing>
```

A reasoned skip is written in the commit body; for the run before the commit
exists, pass the same line with `-reason`. The marker with nothing after it is
red, on purpose.

## `x3 case`, examples that never see the development database

```yaml
case:
  database: isolated   # needs a testdb section; leave it out to keep today's run
```

With `case.database: isolated`, `x3 case` reruns itself inside the `testdb`
wrapper: a fresh copy is cloned, `testdb.dsnEnv`, every `testdb.runEnv` name and
the `testdb.adminDsnEnv` name point at the copy, and the copy is dropped after
the run. A run that is already inside a copy (its `dsnEnv` names a database
carrying the `testdb.prefix` and a creation stamp, as `x3 testdb run -- x3 case`
does) is not wrapped twice. If the copy cannot be built no example runs and the
run is red; nothing is written anywhere. The first stderr line says it:

    x3 case: case.database is "isolated"; the examples run inside a fresh copy, and APP_TEST_DSN, APP_DSN, PG_ADMIN point at it

If the copy cannot be built (here the admin variable is empty) the run stops
before any example, with exit 2:

    x3 testdb: environment variable PG_ADMIN is empty

Why: a narrow `x3 case -only` run without the wrapper handed the shell's
development DSN to the examples, and a test helper that fell back to it wrote
fake rows into the development database. The wrapper existed but had to be
remembered.

Measured against a real Postgres (gate step `case database isolated control
experiment`, fixture `internal/cases/testdata/isolated-db`): an example that
writes one row through the development variable left a development-like
database at 0 rows with the mode on (it wrote into the `x3test_` copy), at 1 row
with the mode left out (today's run, unchanged), and at 0 rows when no copy
could be built (red, nothing ran). The step carries `needs: X3_PG_ADMIN`: on a
machine without that admin connection it is skipped by name
(`skipped: needs X3_PG_ADMIN`) and touches no database.

<!-- x3-dist version=v0.340.0 capabilities=7f596239b33d07b08fcfa0550f33f7a3d4eadd50b8ad99f6fce73dc731635008 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
