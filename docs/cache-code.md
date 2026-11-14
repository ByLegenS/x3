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

<!-- x3-dist version=v0.361.0 capabilities=8f25ee47309c308289d52991abf5d8d9aae99ea0cf44e4eb695f2747b7b306c0 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
