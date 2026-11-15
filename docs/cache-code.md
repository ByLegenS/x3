# A cache record lives as long as the code that wrote it

[The pages](INDEX.md) - [what x3 is](../README.md)

## A cache record lives as long as the code that wrote it

A cached result is valid while the **code that produced it** is unchanged — not
while the binary carries the same release tag. A release that touches only the
screenshot tool keeps every example result, every lint finding and every gate
step that did not run it.

```json
"release": {
  "package": "./cmd/tool",
  "stamp": "main.version",
  "codes": "main.codes"
}
```

`codes` names a string variable of the main package. `x3 release` fills it with
one digest per top-level function of that package: the text of the function and
of every declaration it reaches in the main package, the transitive closure of
every module package it calls (all their source and embedded files, files of the
other operating systems included), the `main` function's own text and the
packages it calls (it runs before every entry), `go.mod`, `go.sum` and the Go
version. A command's slots are salted with its **result** digest (below), never
the release tag; the file name carries no identity. Two releases with the same
`case` code share one record; two with different code write their slots side by
side in one file and never empty each other. The gate keeps its own code out of
its salt: every step carries the code of the engine command it calls in its key,
and a step that calls no engine command (`go vet`) depends on no engine code.

**Comments are not code.** The digest reads a Go file as the compiler does: its
token stream, without comments and without layout (a line break and an explicit
semicolon count the same, so does a semicolon before a closing brace). A release
that only adds inline examples or rewrites a rationale keeps every result.
Comments the toolchain reads stay in: `//go:` (build, embed, linkname),
`//line`, `//export`, `//extern`, `// +build`. A cgo file (its comment is C
source) and a file the scanner rejects are digested byte for byte. Control
experiment: `Codes` over `internal/release/testdata/codes/comment`, the base
module with doc, body and end-of-line comments added and a body split over
lines, gives every entry the base digest; with the token stream turned off
(bytes digested) the same module moves `runA`.

When in doubt it counts more: a name that matches a declaration is followed even
if it is a local, reaching a type reaches all its methods (an interface may call
them), and reaching a variable assigned in `init` reaches that `init`. The price
of counting too much is a cache miss; the price of counting too little would be
the old result of changed code.

A gate step that runs the engine (`x3 case ...`) carries the digest of **that**
command in its key, so a step is re-run when `case` changes even if the gate's
own code did not. A command that wraps another (`x3 testdb run -- x3 case`)
carries the digest of **every** engine command on its line: the wrapper's code
does not answer for the wrapped one. Measured on 2026-11-15 in a two-file
project whose step is `x3 testdb run -- x3 case -only p`: a release changing
only a function `case` reaches left the step cached (`skipped: cached`) when
only `testdb` was keyed, and re-ran it (2,0 s) with both keyed. Control
experiment: the `engineOf` example on a wrapped line goes red with only the
first command keyed; `subcommands` and `engineWord` carry their own examples
(`x3.yaml` is a settings file, not the engine). A non-engine command whose line names `x3` (`bash -c "x3 boxes"`) takes
the digest of the command it names. A command the table does not name (an
alias, a region verb, a flag where the command should be) makes the step
**unrememberable**: it runs every time. It used to take the digest of the whole
engine, or the release tag, and both change on every release (the user,
2026-11-15: *"a cache is not renewed because the version number changed"*). Limit: a program that starts the engine without
naming it on its command line is not seen — the same blindness as the gate's
observation of such a step.

⛔ **No release tag reaches a cache identity.** A binary built without `codes`
(a plain `go build`, the gate's own `{bin}`, a measurement binary stamped only
with a version) salts its slots with **its own digest** (`b<16 hex>`): every
rebuild with changed code is another binary, so an old answer never comes back.
A binary that carries a table but no digest for a cache name is **red** (`the
code table this binary carries names no digest for "graph"`), never a silent
fallback; a result file not named after a command (`graph`) takes the digest of
the function that produces it (`producers` in cmd/x3). The syntax rule
`no-release-version-reaches-a-cache-identity` keeps the tag out of the cache
API in `cmd/x3`, `internal/cache` and `internal/gate`. Control experiments: the
`Tag`, `Code` and `engineOf` examples; the syntax rule goes red on
`func Stamp(` written back into internal/cache.

**Two settings, one file.** Every working copy of a project shares one cache
directory, and two working copies opened from different commits can carry
different settings — so different salts for the same command. The salt is part
of every slot's hash, not a single field of the file: a working copy whose salt
does not match misses the other's slots but leaves them in place, and an
unasked slot ages out after `stale` runs like any other. Keys a command's result
does not depend on stay out of its salt: `x3 case` drops `case.workers` (how
many packages run side by side) and `case.deferred` (the verdict on examples
that did not run; a package with a deferred example is never stored).

Measured in the pilot on 2026-11-15, two small projects sharing one cache
directory and differing only in `case.workers`: with the old binary the first
project's third run found no record (the second project's run had emptied the
file) and the second project ran cold; with the new binary both hit.

## A record is weighed by the code that produces results

The `codes` digest of a command covers everything it reaches, including how its
cache is stored. A release that changes only the cache package or the command
table would get a new `case` digest and a cold run, although no example could
pass or fail differently. `release.results` names the part of the code that
produces the record; that digest salts the record's slots, so a release whose
digest of that part is unchanged reads the same slots.

```yaml
release:
  package: ./cmd/x3
  codes: main.codes
  results: main.results     # string variable of the main package
  scopes:
    - command: case          # the command whose cache inherits
      entry: runCase         # its function in the main package: a root, and its cache identity
      package: ./internal/cases
      roots: [Config.Check, Config.runDart, Config.runScripts]
      with: [./internal/testdb]   # counted whole
      skip: [./internal/cache]    # never counted: builds keys and stores
```

The digest is declaration-level **across the module** (the same walk as
`release.Reaches`): the text of every declaration the roots reach, in `package`
and in every module package they call, the own text of each `with` package (its
declarations are roots, so what it calls counts at declaration level too), the
declarations `entry` reaches in the main package (`main()` and the main
package's init are not counted), `go.mod`, `go.sum` and the Go version. A `skip`
package is left out: code that only builds keys and stores records cannot change
what a record says, and a key that changes simply misses. A root or entry that is
not declared is an error, never a smaller digest. It used to count the whole
package closure of every package the runner touched and of the `with` package:
measured on 2026-11-15, a changed body in the settings package that the runner
never calls (`config.Reads`, used by the gate's timing line) moved `case@self`
and dropped every example record; it now keeps it (build `d` in the table under
"One result file per command"). Control experiments: the `ResultCode` examples
on `testdata/results/dbidle` (an unreached declaration added to a called
package) and `withidle` / `withname` (an unreached and a reached body in a
package the `with` package calls); with the old closure, `dbidle` and `withidle`
go red.

The roots are the whole runner, not only the run: the code that **finds**
examples is inside, because a package's key is its bytes, not the rules that
read them — a release that recognises a new example form would otherwise inherit
the package's old green and never run the new example. Recalling examples from
the per-example record is inside too: a recalled example's green is part of the
package green.

`x3 release` embeds `case@self=<digest>`; the run salts its slots with it
(`cache.Code`). A changed setting builds a different salt and misses; a binary
without the table salts with its own digest. `x3 release -stamps` prints the
`-X` flags a tree would embed, so a test binary can carry the release's table.
The table carries only this tree's rows (`case@self`); past releases' rows are
not embedded, because records live in one file and are keyed by the code that
produced them.

Control experiments: the `ResultCode` examples in internal/release/results.go
(`testdata/results`: a planning-only change keeps the digest; a changed body the
root reaches, a changed `with` package, a changed body the entry reaches move it;
a changed `skip` package keeps it, the same change without `skip` moves it; a
missing root or entry is an error) and the `Open` examples in
internal/cache/cache.go (same salt in one file: found; different salt: missed,
the other's slot kept). With the `skip` deletion turned off one example goes
red.

**Every command has its own result row, at declaration level.** A command with
no declared scope inherits too: the release embeds, for every entry point of the
binary, the digest of **the declarations it reaches across the whole module**
(`release.Reaches`): a call into another package goes to the named declaration,
not to the whole package; a reached type brings its exported methods; the init
functions of every imported package count, `main()` and the main package's own
init (it only builds the command table) do not; embedded files and
`go.mod`/`go.sum`/the Go version count. `release.skip` names packages that never
shape a result (the engine writes `skip: [./internal/cache]`). The rows ride in
the same table as `fn:<entry>@self`; the command line
renames them to the command (a declared scope's row wins). A gate step that
calls a command is now keyed by that command's **result** code (`cache.Code`),
so a release that leaves a checker's reached code alone keeps both the checker's
records and the gate steps that call it.

```text
engine, 168 entry points, last 6 releases:
  same result code in all 6: 57   two codes: 11   changed every release: 8
  (package-closure code: changed every release for all 168)
```

Control experiments (`internal/release/reach.go`): an unreached method body
changes and the entry's code holds (the package closure moved); a reached body,
a reached function in another package or the entry's own helper changes it;
another entry's body does not; a skipped package is not counted - with the
cross-package calls left out the two cross-package arms are red. `cache.Code`:
a command with a result row answers `r<code>`, one without answers its code.


### A release that changes only recall code keeps the greens that really ran

The per-example record is written in two parts with one copy per example: the
**pure** part (`<dir>#examples#pure`, `<dir>#ran#pure`) holds examples whose
green came from a real run — they ran in this run, or they were recalled with
exactly the key a pure record already held for them — and the rest holds
examples recalled by a judgement (body-only change, new function in an imported
directory). A run reads both and joins them. A second scope row names the
runner minus that judgement:

```yaml
    - command: case#pure
      entry: runCase
      package: ./internal/cases
      roots: [Config.Check, Config.runDart, Config.runScripts]
      except: [Config.recall, Config.judge, Config.untouched]   # never walked into
      with: [./internal/testdb]
      skip: [./internal/cache]
```

`except` stops the walk at those declarations: their text, and whatever only
they reach, is left out of the digest; a name that is not declared is an error.
Keys ending in `#pure` are salted with the `case#pure` digest (`Cache.Pure`),
every other key with the `case` digest: a release that changes only the recall
judgement still finds **only the `#pure` keys**; every other key misses. A recalled green is the work of the old judgement, which this
release may have changed; a green that really ran depends only on the code that
ran it. The code that writes the record and builds keys stays in the digest or
changes the key, and a changed key misses.

Control experiments: the `Pure` examples in internal/cache/cache.go (same
`#pure` salt, different plain salt: the `#pure` key is found, a plain key is
not; different `#pure` salt: nothing), the `ResultCode` examples with `Except` (a changed
excepted body keeps the digest, a changed `with` package still moves it, an
unknown name is an error) and `pureEntry`, `joinMemo`, `joinBook` in
internal/cases. With the suffix filter, the `except` stop or the head match
(below) turned off, one example each goes red.

### Records are picked by the run's head, not by age

A package's records sit under one cache key each, newest first. Working copies
share the cache directory, and one with other settings (another migration set
behind the test database template, another timeout) writes records with another
head; a run that read the newest record then refused every per-example recall
with `the run's settings or toolchain changed`. A run now reads the newest record
written with **its own** head and falls back to the newest one only when none
exists, where a different head still means the static rule.

Measured in the pilot on 2026-11-15: in one cache file the coverage record of
the same package alternated between three heads within one release, and a
changed package reran 2194 of its 2590 examples for that reason. Control
experiment: the `ofHead` examples in internal/cases/memo.go (two slots with two
heads: today's head is returned where the old read returned the newest; no slot
with today's head: the newest).

Measured in the pilot on 2026-11-15: six releases in one night, each followed by
the first gate rerunning 37 000 examples cold (54 minutes), because the salt was
the release tag. The cache directory itself was already shared by every working
copy (`~/.x3cache/<name>`) and keys are relative paths, so a new working copy
with the same content finds the same records.

Control experiments: `Codes` over four small modules in
`internal/release/testdata/codes` — a change in a package one entry does not
reach leaves that entry's digest alone and moves the other's; a change in a
package it reaches moves it; a new entry in an `init`-built table moves the entry
that reads the table and no other. `tagOf` and `Open` in `internal/cache`: the
same code under a new release finds the old green, different code does not and
does not empty the old file. End to end, four binaries over one small project
sharing one cache directory: the same codes under a new tag hit (`x3 case`: 1 hit,
0 misses; `x3 gate`: the step skipped as cached); a changed `runCase` digest misses
in both, although the gate's own digest did not move; a binary without codes
misses (the old behaviour); the project copied to a second path hits.

<!-- x3-dist version=v0.382.0 capabilities=61adc1f94060c322e08970daff7cbc68aeea6dd9543a0b354b5a35120c880b12 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
