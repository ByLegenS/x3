# Making the release itself

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 release`

```
x3 release [-config <file>] [-root <dir>] [-allow-dirty] [-out <file>]
```

**Catches:** a publication assembled by hand — a binary nobody can rebuild, a page
announcing one version next to a page announcing another, a document that went out
without passing the gate that measures it.

A release is usually a shell script, and a shell script is where a release chain
goes to rot: every project that adopts the engine writes the same one again, and
each copy drifts. This engine kept such a script for a long time — 844 lines of
PowerShell across two files — and deleted it the day it could no longer say why a
project should not have to write one.

### The binaries, built twice

Each target is compiled, then compiled **again** into a temporary path, and the two
checksums are compared. One build plus a checksum only produces the number the
downloader will verify; a toolchain that does not give the same bytes twice would
pass that check in silence.

```json
"release": {
  "dir": "../publication",
  "package": "./cmd/tool",
  "stamp": "main.version",
  "targets": [
    { "os": "windows", "arch": "amd64", "name": "tool-windows-amd64.exe" },
    { "os": "linux",   "arch": "amd64", "name": "tool-linux-amd64" }
  ]
}
```

The build flags are the reproducibility: `-trimpath` drops the path the build ran
from, `-buildvcs=false` the repository stamp, `-buildid=` the compiler's own id,
and `CGO_ENABLED=0` the host's C library. The only thing left varying is the
version string, which is the point of `stamp`.

`SHA256SUMS.txt` is written from what was actually built. `LATEST` is written
**only** for a clean tagged tree: a build from `v1.2.3-2-gabc123` is a test build,
and a pointer naming a version nobody can check out is worse than no pointer.

### The pages, generated from one source

One document carries the whole text; markers split it. Each marker is an HTML
comment: `x3:doc <slug> | <title> | <summary>` opens a page, `x3:docindex` is
where the roster of pages is written, and `x3:ref <title>` above a table moves
that table out to its family's reference page, leaving a link behind. A marker
that survives into a generated page is red - it would have been published as
text nobody meant to write.

```json
"pages": {
  "source": "docs/CAPABILITIES.md",
  "template": "docs/README.tmpl.md",
  "back": "what the tool is",
  "gates": ["tools/public-gate.json"]
}
```

Every generated document carries the same stamp — the version, and the checksums of
the source and the template — so *"these two pages describe different versions"*
cannot happen: they were written in one run or not at all. Anchors are followed
across the split: a link to a heading that moved to another page is rewritten to
its new home, a link to an anchor **no** published document defines is red, and so
is an anchor two documents define, because the rewrite would have to guess.

### The gates run before anything is written

The generated text is staged first, the gates named in `gates` run **on the staged
tree**, and only a green run reaches the release directory. Writing first and
reverting on red would leave the publication wrong for as long as the gate takes.
On red the staged tree is kept, named in the message, so the text that failed can
be read rather than regenerated.

Each gate file brings its own sections: a `secrets` section measures the generated
text for leaks (a local path, a home directory, an address) and, with a second
file, for language; a `freeze` section holds each published document under its own
line cap. A page with no cap declared is not a page this repository publishes.

<!-- x3-dist version=v0.180.0 capabilities=ecb1c78593b9d424c2fc8f64fef7ec5045f07d669c43fc94ae2bfe643ccd47a3 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
