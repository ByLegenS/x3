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

<!-- x3-dist version=v0.276.0 capabilities=b5f0a4a130a3904a44ddab3b5648bead1cce4e08a839e806253889f72d5f3786 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
