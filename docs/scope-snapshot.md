# A change read without git

[The pages](INDEX.md) - [what x3 is](../README.md)

## A change read without git

Every gate that narrows its work first asks *what changed*. Until now all three
answers went to git: `head` reads the last commit, `working` reads the dirty
tree, `auto` picks between them. `-scope snapshot` asks the engine's own record
instead — the `changed`, `new` and `gone` paths of the snapshot above:

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

## One resolver opens the declared cache directory

`cache.dir` may be written as `~/.x3cache/<name>` or `${VAR}/cache`, and
exactly one function opens it — `cache.Dir`. Everything that puts a file under
the declared directory asks that function, including the machine tuning record
(`gate-tune.yaml`).

This is a rule because it was broken. Measured in the pilot on 2026-09-21: the
tuning record joined the *raw* string, so a project declaring `~/.x3cache/vt`
grew a directory literally named `~` at the root of its working tree while the
main cache sat correctly in the home directory. One declaration, two
destinations. A `.gitignore` pattern hid the litter from git, so nothing but a
file browser could see it.

<!-- x3-dist version=v0.288.0 capabilities=b7fca59edbf65759483bfdca34f14aeafbe84562986ae2f4e8a4b427249da8fb template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
