# A change read without git

[The pages](INDEX.md) - [what x3 is](../README.md)

## A change read without git

Every gate that narrows its work first asks *what changed*. Until now all three
answers went to git: `head` reads the last commit, `working` reads the dirty
tree, `auto` picks between them. `-scope snapshot` asks the engine's own record
instead — the `changed`, `new`, `gone` and `moved` paths of the snapshot above:

```
x3 scope  -scope snapshot     # the lanes, measured against the last run
x3 docs   -scope snapshot
x3 test   -scope snapshot
x3 mutate -scope snapshot
```

It is a **different question**, not a cheaper route to the same one. Git says
"this commit touched these files"; the snapshot says "**these files have moved
since the gate last measured them**" — which is what a gate actually wants to
know, because a gate is not judging a commit, it is judging whether its own
measurement is still fresh.

The control experiment (`snapshot scope control experiment`) runs one lane gate
over one planted edit, six arms:

| Arm | Reads | Result |
|---|---|---|
| nothing moved | snapshot | lane quiet, exit 0 |
| a file edited outside the lane | snapshot | `outside: outside/b.txt` |
| the same edit, `env: {PATH: ''}` | snapshot | **the same file, same line** |
| the same edit, `env: {PATH: ''}` | `working` | `git is not on this machine` |
| a file that never existed | snapshot | `outside: outside/new.txt` |
| a file the disk no longer holds | snapshot | `outside: outside/gone.txt` |

Changed, new and gone — all three, with git off the machine.

**One thing the snapshot cannot answer, named.** The cache keeps digests, not
contents, so a snapshot-scoped run has no "before" to read. The exemption that
lets `x3 test` skip a unit whose only change was a *comment*
(`affected.Config.prose`) needs the previous text and therefore still asks git:
under `snapshot` it simply does not fire, and the unit runs. The direction is
the safe one — a run that measures too much costs time, a run that measures too
little reports a green it never took.

### It reads a change without git by default

Measured in the pilot on 2026-09-21: the gate itself never calls git, but
*selecting the tests* did — `x3 test` reaches `changed.Read`, and that was the
one path a daily run could not take on a machine without git. It now defaults
to the snapshot:

```
x3 scope                      # -scope snapshot, the default
x3 scope -scope head          # git, by choice
```

`head`, `working` and `auto` are all still there and all still read git. What
changed is which one you get when you say nothing.

**A tree no run ever remembered does not report an empty change.** That is the
trap this default could have walked into: an empty list reads as "nothing
changed", every gate passes, and the green means nothing. The source says the
absence out loud instead:

```
BLOCK : no_repository
	cannot read what changed: cache/gate.yaml remembers no path of this
	tree, so there is no "since the last run" yet: run the gate once, or
	ask git with -scope head or -scope working
```

The control experiment (`gitless default scope control experiment`) has four
arms: the default names the file with `env: {PATH: ''}`; the *same command*
with git installed prints the same line; a second gate (`x3 docs`) reads the
same default; and a tree with no record says the sentence above instead of
passing quietly.

**Inside the gate, a missing record is not a verdict.** A gate step that runs
`x3 docs` (snapshot scope) or `x3 boxes` (with `suspect.gone`) reads the gate's
own record, and that record is written at the *end* of a run. On a cache that
remembers nothing - a new home directory, or the first run after any release,
since the record's name carries the version - those steps said the sentence
above and went red; the record lives inside the cache, outside the step's key,
so the red was remembered and every later run replayed it. Measured on a pilot
(2026-10-02): the same tree was green on a shared cache and red with two
"born" findings on a cold one, with both v0.293.0 and v0.294.0.

The gate now notices a cache that remembers no run, writes its own record once
the steps are done, and measures the red steps **once more**. Green steps do
not run again; a red that is real stays red. The run says so:

```
-- this cache remembered no run, so 2 red step(s) were measured again once the gate had written its own record: open work + generated documents, document follows code
```

Note that the record is named per version: a step that calls an `x3` of a
*different* version than the gate reads a different record. Run the gate and
its steps with the same binary.

### A path that was here once

A criterion that says *"this file is gone"* can only prove it holds by losing
once: the path was there, the work removed it, the criterion turned green. Spell
the path wrong and that day never arrives — `push_tst.go` is met the moment it is
written and can never fail again. Measured in the pilot: sixteen criteria of this
class, not one of them guarded.

The question is about the **past**, so it cannot be put to today's tree. It is
put to the snapshot above — the paths the gate's own cache remembers reading:

| The path | The record | On disk | The verdict |
|---|---|---|---|
| `here.txt` | remembers it | still there | **red** — a closed box without proof |
| `left.txt` | remembers it | gone | **green** — the day the criterion was written for |
| `lfet.txt` | never saw it | gone | **red** — check the spelling |

```yaml
boxes:
  file: work.yaml
  suspect:
    gone: true        # put every gone-criterion to the record
cache:
  dir: cache          # the record is the gate's own cache
```

**Two absences, two sentences, and no silent green in either.** A tree that
declares no cache is told that; a tree whose cache remembers nothing is told to
run the gate once. Neither passes quietly, because a rule that cannot read the
record would call every gone-criterion a typo:

```
x3 boxes: boxes: suspect.gone is written and x3.yaml declares no cache; the
question "was this path ever here" is answered from what the gate remembered,
so write cache.dir and run the gate once
```

**The limit, and which way it errs.** The cache prunes slots that have gone
stale, so a path last read long ago can be forgotten. A forgotten path moves from
*gone* to *unknown*, which is more red and never less: the criterion asks to be
re-read rather than passing on a memory nobody holds.

The control experiment (`gone without git control experiment`) runs five arms on
one planted tree — the three rows of the table, then the green row and the
misspelled row again under `env: {PATH: ''}`, reaching the same verdicts with no
git on the machine.

### An exclusion list x3 reads itself

Some paths a work list names were never measured by anybody: build output, a
generated file, a whole excluded tree. They are dropped from the question — and
**counted** as they go, because a criterion eliminated in silence cannot be told
from one nobody wrote. `summary.untracked` is that count.

Which paths those are is already written in the repository, in `.gitignore`, in
plain text. The engine reads that file itself: it walks from the root down, reads
a list in every directory above the path, and lets the last matching line win —
the deeper list overriding the shallower one, and an excluded directory keeping
its children excluded even against a later `!`.

| The line | Reads as |
|---|---|
| `build/` | a directory, anywhere below this list |
| `/build` | `build` at this list's own level only |
| `*.exe` | that suffix at any depth |
| `a/b.txt` | one path, relative to this list |
| `!keep.tmp` | taken back out of the exclusion |

**What it does not read, named:** character classes (`[a-z]`) and backslash
escapes (`\#`, `\!`). An unread line matches nothing, so the path stays in the
measurement — more red, never less.

The control experiment (`own exclusion control experiment`) puts two closed boxes
in one tree, one resting on `build/out.bin` and one on `keep.tmp`. With the list
in the tree only `keep.tmp` is named; under `env: {PATH: ''}` the same two paths
are classified the same way; and in a tree carrying **no list at all**
`build/out.bin` is named too — so the difference came from the list and from
nothing else.

## A step remembers the configuration sections it consulted

When a gate step calls an engine command, that command reports which
**sections** of the configuration it consulted, each with a digest of the
section's meaning (comments and layout stripped). The gate stores those
digests with the step and answers from the cache while every one of them
still holds. A comment dropped into `gate:` therefore re-runs nothing, an
`arch:` rule that changes re-runs only the steps that consulted `arch`, and a
change with a meaning in `gate:` **outside the step list** (`slow`, `env`,
trees, regions) still moves the gate salt and re-runs every step. The step
list itself is not in the salt: each step carries its own definition in its
key, so editing or adding one step re-runs only that step. A rule that reads the configuration as data (`from: x3`) consults only
the sections its selection reads (`gate:runs` reads `gate`).

A trial's or step's `env` carries the same `{bin}` / `{tmp}` placeholders as
its run line; `{bin}` becomes the absolute path of the binary under test, so a
tree built by an experiment can call the engine it measures:

```yaml
- name: config section memory control experiment
  env:
    X3_UNDER_TEST: '{bin}'
  trials:
    - run: '{bin} gate -config {tree}/x3.yaml -root {tree} -cache {tmp}/memory/sections.yaml -workers 1'
      tree: config-sections:commented
      says: 0 ran, 2 skipped
      want: 0
```

Measured on the same tree (two steps calling `arch` and `lang`): base twice
gives `2 ran` then `0 ran`; a comment in `gate:` gives `0 ran, 2 skipped`
(the engine before this change: `2 ran, 0 skipped`); an `arch` rule change
gives `1 ran, 1 skipped` (before: `2 ran`); changing one step's own
definition gives `1 ran, 1 skipped` (before: `2 ran`); a meaningful `gate:`
key outside the steps (`slow: 2999`) gives `2 ran` on both.

**An overlay run is held by its section too.** A command run with
`-with <file>[:<name>]` records the consulted section together with the
overlay, and the gate re-resolves the section under the same overlay to weigh
it; the overlay file itself stays an input by its bytes. Before, an overlay run
bound the step to the bytes of every configuration file — measured on a pilot,
one comment in its gate settings re-ran 22 overlay trials.
`config overlay memory control experiment` (one step running
`arch -with overlay.yaml`): base `1 ran`; a comment in `gate:` gives
`0 ran, 1 skipped` (the engine before this change: `1 ran, 0 skipped`); the
overlay moving gives `1 ran`; the rule under the overlay moving gives `1 ran`.

**Which file a key lives in is held by meaning too.** A command that asks in
which settings file a rule is written (`placement`, `adoption`) records the
layout of the configuration — each file's name and the meaning of its keys —
and not the bytes of every file. A comment in a settings part re-runs neither;
a section moved to another file does. `x3 fmt` still reads the bytes, because
there the bytes are the answer. `config layout memory control experiment`
(a placement step over a split configuration): base `1 ran`, again `0 ran`; a
comment in the gate part `0 ran, 1 skipped` (the engine before this change:
`1 ran`); the placement section moved into its own file `1 ran`.

**An inherited scope does not count the configuration as a source.** A step
that declares no `touches` inherits the gate's `reads`, and that list usually
names the settings directory. The gate's own settings files (the root and every
part) are dropped from such a step's inputs: their meaning already sits in the
step's key and in the sections the step consulted. A step that really reads a
settings file as a source says so — by its observation when it calls the
engine, or by writing its own `touches`, which this filter never touches.
Measured on a pilot: 76 of 97 steps inherited the scope, and one comment in the
gate settings re-ran every step that calls an outside tool.
`config inherited scope control experiment` (one step running `go version`,
scope inherited): base `1 ran`, again `0 ran`; a comment in `gate:`
`0 ran, 1 skipped` (before: `1 ran`); a source under the scope moving `1 ran`.

**A walked directory's name list leaves out what nobody measures.** The list a
step is held by drops every entry the repository's ignore list names (read by
the engine itself, see *An exclusion list x3 reads itself*) and the run's own
cache directories. Measured on a pilot: the first run in a fresh working copy
opened its own scratch directory, and the next run re-ran because its parent
changed. `walked directory listing control experiment`: base `1 ran`; an ignored
scratch directory appearing `0 ran, 1 skipped` (before: `1 ran`); an ignored
build output `0 ran, 1 skipped`; a file nobody ignores `1 ran`.

The cache must not sit in a directory above the tree: the
cache's own directory is excluded from what a step is said to read.

A configuration file that does not exist is consulted too, and its answer is
one answer whatever path named it: `x3 case -config t/none.yaml` records the
absent file, and the gate later weighs it under the root's absolute path
without re-running the step. `x3 snapshot` (and the `x3 docs` / `x3 scope`
runs that read it) holds a consulted section by its answer, the way the gate
does: a file that was absent when consulted and is still absent has not
changed, and a file counts as gone only when its answer moved and it is no
longer on disk. Measured on this repository: before, `x3 docs` reported
`internal/move/testdata/dry/x3.yaml` and `internal/cases/testdata/none.yaml`
as changed code (neither file ever existed) and the docs step was red.

## One resolver opens the declared cache directory

`cache.dir` may be written as `~/.x3cache/<name>` or `${VAR}/cache`, and
exactly one function opens it — `cache.Dir`. Everything that puts a file under
the declared directory asks that function, including the machine tuning record
(`gate-tune.yaml`).

This is a rule because it was broken. Measured in the pilot on 2026-09-21: the
tuning record joined the *raw* string, so a project declaring `~/.x3cache/app`
grew a directory literally named `~` at the root of its working tree while the
main cache sat correctly in the home directory. One declaration, two
destinations. A `.gitignore` pattern hid the litter from git, so nothing but a
file browser could see it.

<!-- x3-dist version=v0.337.0 capabilities=3860e842c699cce7f98e5bd335013a1d4b84da590e84edf414492455d8caaf11 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
