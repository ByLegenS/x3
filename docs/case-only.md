# A run narrowed to one file or one example

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, a run narrowed to one file or one example

**What it is for:** a control experiment. The proof that an example measures
anything is that **one** of its propositions falls when the code it reads is
broken — and that arm has to be cheap enough to write. Without narrowing its
price is the price of the whole **package**: measured on `internal/ledger`
in the pilot repository, 982 examples, 25-60 s per arm with a database attached.
Ten arms is then ten minutes of waiting, and a ten-minute experiment does not
get written.

```
x3 case -only <target>[,<target>...] [dir]
```

A **target** is a file, which selects every example that file carries, a
`file:line`, which selects the **block** standing on that line, or a directory,
which selects its package (below). The path may be
written from the measured root or as the bare file name, whichever tells the
files apart; a name matching two files is an error rather than a silent choice
of the first.

A path may also be written **the way the shell sees it** when the root is a
directory elsewhere: `x3 case -config <tree>/x3.yaml -only <tree>/a.go <tree>`
reads `<tree>/a.go` as the root's `a.go`. Only a target that stands on disk and
falls inside the root is rewritten; one outside it stays as written and is the
same unknown-target error as before (exit `2`, `no file the run reads is named`).
Until 2026-10-03 every such target was unknown, so a gate trial on a planted
tree (`{tree}`, outside the repository) could not narrow.

```
$ x3 case -only calc.go:13 ./only          # a line of prose inside Sub's block
SELECTED calc.go:14 (Sub)
x3 case: 1 of 4 example(s) selected - 1 passed, 1 fixture file(s), 0 finding(s)

$ x3 case -only calc.go ./only             # the file's own two examples
SELECTED calc.go:8 (Add)
SELECTED calc.go:14 (Sub)
x3 case: 2 of 4 example(s) selected - 2 passed, 1 fixture file(s), 0 finding(s)

$ x3 case -only mul.go:8 ./only            # one block, two examples - both
SELECTED mul.go:8 (Mul)
SELECTED mul.go:9 (Mul)
x3 case: 2 of 4 example(s) selected - 2 passed, 1 fixture file(s), 0 finding(s)
```

**The line may be any line of the block** — the directive itself, the prose
beside it, or the declaration the block sits on. Only the directive's own line
would have meant that the line an author reads in the editor has to be the line
the engine counts, and an off-by-one there produces an error instead of a run.
The block is also the **unit**: two examples written above one declaration are
selected together, because they prove one declaration.

**Every selected example is named**, one `SELECTED` line each. A count alone
would tell an author who mistyped a target that something was measured without
telling them what.

**An unknown target is an error (exit `2`), never an empty green run**, and the
two ways of being unknown are two different sentences:

```
$ x3 case -only nosuch.go ./only
x3 case: -only nosuch.go: no file the run reads is named nosuch.go; 2 file(s) carry examples

$ x3 case -only calc.go:999 ./only
x3 case: -only calc.go:999: calc.go carries 2 example(s) and none of them stands on line 999; the blocks are at 5-9, 11-15
```

**A directory selects a package.** Every example-carrying file of that one
directory is chosen — a migration round no longer has to count a package's files
by hand. Subdirectories are **not** included: in Go a package is one directory,
and a target that took everything below it would run packages nobody named.

```
$ x3 case -only left ./only-dirs             # left/deep is another package
SELECTED left/left.go:7 (Twice)
x3 case: 1 of 3 example(s) selected - 1 passed, 0 finding(s)
```

**A file the run reads but which carries no example** — a fixture, a test file —
is skipped with a `NOTE`, not an error: it is compiled with its package anyway,
and an author listing a package's files should not have to weed it out. This
hides no wrong selection, because three things still hold: the skipped file is
named; a target that names no file the run reads is still exit `2`; and a
narrowing whose every target was skipped measures nothing, which is still exit `2`.

```
$ x3 case -only calc_fixture.go,mul.go ./only
NOTE calc_fixture.go:0 (calc_fixture.go): only_target_carries_no_example
x3 case: 2 of 4 example(s) selected - 2 passed, 1 fixture file(s), 0 finding(s)

$ x3 case -only calc_fixture.go ./only
x3 case: -only calc_fixture.go: no target selects an example; a narrowed run that measures nothing is not a green one
```

⛔ **A narrowed run neither reads nor writes the cache.** Reading it would let a
stored verdict answer for an example the author asked to measure; writing it
would let the next full run count 980 never-measured examples as passed. The
cache is the one mechanism that can present an unmeasured green as a measured
one, and what would feed it is exactly a narrowing that stays quiet:

```
$ x3 case -cache c.yaml -only calc.go:9 ./only   # fresh cache, narrowed
x3 case: 1 of 4 example(s) selected - 1 passed, 1 fixture file(s), 0 finding(s)

$ x3 case -cache c.yaml ./only                   # the full run measures anyway
x3 case: cache 0 hit(s), 1 miss(es)

$ x3 case -cache c.yaml ./only                   # and only a full run stores one
x3 case: cache 1 hit(s), 0 miss(es)
```

**Narrowing narrows the run, not the reading.** Every file is still parsed and
every declaration law — a dead import, a dead type, a dead build tag, a
declaration in a fixture nobody names — is still asked of the whole tree. A
narrow run answers its own question; it does not silence anybody else's. The
package still compiles whole, too: its fixture and its sibling `_test.go` files
enter the same build, and only the **generated** test carries fewer lines.

**What narrowing does not remove is the compile.** Measured on this engine's own
`internal/cases`, warm build cache: 54 examples in **7.0 s**, narrowed to 5 cheap
examples in **1.2 s**, and narrowed to the 12 examples of `Config.Check` — each
of which runs a Go toolchain of its own — in **7.1 s**. The floor is the
package's own build; what a target buys back is the time of the examples it left
out.

<!-- x3-dist version=v0.313.0 capabilities=17d7c952c83d131183aa1ec638aa096e3ac52de533b3d0cea3c70480d6a35004 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
