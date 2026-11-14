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

<!-- x3-dist version=v0.363.0 capabilities=5af223242f5e9eea4e5eac02f01d16f305004dc2734004f5f313971d7d9edb72 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
