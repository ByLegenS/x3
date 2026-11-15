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
version. A command's result file is then named after that digest instead of the
release tag, and the same digest is its salt. Two releases with the same
`case` code share one record; two with different code write two files and never
empty each other (the reason result files were split per version is kept).

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
own code did not. A command the table does not name (an alias, a region verb),
and a non-engine command whose line names `x3` (`bash -c "x3 boxes"`), take the
digest of the whole engine. Limit: a program that starts the engine without
naming it on its command line is not seen — the same blindness as the gate's
observation of such a step.

A binary built without `codes` (a plain `go build`, the gate's own `{bin}`)
falls back to the release tag: no digest, no sharing, never a stale result.

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

## A release inherits the record when the code that produces results is unchanged

The `codes` digest of a command covers everything it reaches, including how its
cache is stored. A release that changes only the cache package or the command
table gets a new `case` digest, a new result file and a cold run, although no
example could pass or fail differently. `release.results` names the part of the
code that produces the record; a release whose digest of that part equals a past
release's inherits that release's record.

```yaml
release:
  package: ./cmd/x3
  codes: main.codes
  results: main.results     # string variable of the main package
  inherit: 6                # past release tags whose digest is embedded
  scopes:
    - command: case          # the command whose cache inherits
      entry: runCase         # its function in the main package: a root, and its cache identity
      package: ./internal/cases
      roots: [Config.Check, Config.runDart, Config.runScripts]
      with: [./internal/testdb]   # counted whole
      skip: [./internal/cache]    # never counted: builds keys and stores
```

The digest is declaration-level: the text of every declaration the roots reach
in `package`, the closure of every module package those declarations call, the
`with` packages, the declarations `entry` reaches in the main package and the
packages it calls (`main()` is not counted), `go.mod`, `go.sum` and the Go
version. A `skip` package is left out with its whole closure: code that only
builds keys and stores records cannot change what a record says, and a key that
changes simply misses. Reaching a type reaches only its exported methods; the
receiver type of a root reaches none (the runner's methods are followed only by
name). A root or entry that is not declared is an error, never a smaller digest.

The roots are the whole runner, not only the run: the code that **finds**
examples is inside, because a package's key is its bytes, not the rules that
read them — a release that recognises a new example form would otherwise inherit
the package's old green and never run the new example. Recalling examples from
the per-example record is inside too: a recalled example's green is part of the
package green.

`x3 release` embeds `case@self=<digest>` and, for each of the last `inherit`
release tags, `case@c<code>=<digest>`, where `c<code>` is that release's cache
identity (its result file name), recomputed from the tag's tree. When a run opens
`case@<own>.yaml` and misses, it reads the newest sibling whose identity carries
the same digest and has results, rebuilds that release's salt from the same
settings, and answers from its slots; every answer it takes is written back under
the new salt, so the next run reads its own file. A changed setting rebuilds a
different salt and nothing is inherited; a tag whose digest differs, or a binary
without the table, inherits nothing. The run says so on stderr:
`x3: cache: case@c….yaml inherited N record(s) from case@c….yaml: the code that
produces them did not change`. `x3 release -stamps` prints the `-X` flags a tree
would embed, so a test binary can carry the release's table.

Control experiments: the `ResultCode` examples in internal/release/results.go
(`testdata/results`: a planning-only change keeps the digest; a changed body the
root reaches, a changed `with` package, a changed body the entry reaches move it;
a changed `skip` package keeps it, the same change without `skip` moves it; a
missing root or entry is an error) and the `heirsOf` examples in
internal/cache/heir.go (same digest: inherited and rewritten under the new salt;
different digest, different settings, no table: nothing inherited). With the
`skip` deletion turned off, and with the digest comparison turned off, one
example each goes red.

**Every command has its own result row, at declaration level.** A command with
no declared scope inherits too: the release embeds, for every entry point of the
binary, the digest of **the declarations it reaches across the whole module**
(`release.Reaches`): a call into another package goes to the named declaration,
not to the whole package; a reached type brings its exported methods; the init
functions of every imported package count, `main()` and the main package's own
init (it only builds the command table) do not; embedded files and
`go.mod`/`go.sum`/the Go version count. `release.skip` names packages that never
shape a result (the engine writes `skip: [./internal/cache]`). The rows ride in
the same table as `fn:<entry>@self` and `fn:<entry>@c<code>`; the command line
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
When a run finds no sibling with the same `case` digest but one with the same
`case#pure` digest, it inherits **only keys ending in `#pure`** from it; every
other key misses. A recalled green is the work of the old judgement, which this
release may have changed; a green that really ran depends only on the code that
ran it. The code that writes the record and builds keys stays in the digest or
changes the key, and a changed key misses.

Control experiments: the `heirsOf` examples (same `#pure` digest, different
`case` digest: the `#pure` key is inherited, a plain key is not; different
`#pure` digest: nothing), the `ResultCode` examples with `Except` (a changed
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

<!-- x3-dist version=v0.379.0 capabilities=8e76f4cd9d32eef7b90f8c742f220ef113f3d57ecb0e0ed0ac8aaf47c6767ea1 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
