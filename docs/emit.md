# Writing the documents the engine measures

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 emit`

```
x3 emit [-config <file>] [-only <names>] [-from <name=file>] [-check] [-out <file>] [dir]
```

**Catches:** the generated document that quietly stops matching what it was
generated from. Every project that keeps a work list, a plan, or a status page
ends up writing a script that walks the tree, groups what it finds, and prints
markdown. Measured in a production repository: three of them, 1 191 lines, all
doing the same three things — scan, group, write — and the scan was a **second
copy** of a scan the engine already runs.

The split is: the **measurement** is the engine's, the **shape** is the
project's. The engine cannot know what a document should look like, so the
template lives in the repository. What the engine brings is the three things
every hand-written generator was missing.

```yaml
emit:
  documents:
    - name: open-work
      out: docs/OPEN-WORK.md
      template: docs/templates/open-work.md.tmpl
      reason: one list of every unchecked box, so nobody reads twelve files
    - name: evolution
      out: docs/PLAN.md
      template: docs/templates/plan.md.tmpl
      data:
        steps: docs/data/steps.txt
        map: docs/data/map.txt
```

The report a template reads is **any command's `-out` file**, handed over on the
command line: `x3 emit -from open-work=/tmp/boxes.yaml`. The engine imports no
command here — the template sees the report's **YAML keys**, which are already a
published surface, not the engine's Go field names. `from:` in the settings is
the same thing for a fixed path, and the flag overrides it.

### The three things a hand-written generator is missing

**It cannot go stale.** `-check` writes nothing and asks whether the file on
disk is what the template produces today; a difference is `stale_document`, and
it is a `block` by default. A generated document nobody regenerates is worse
than a missing one: it is read as current.

**The template can raise a finding.** A document generator is usually a gate as
well — "this box has no place in the order" is a question only the side that
builds the shape can ask. `{{finding "..."}}` writes it in the engine's finding
language (`template_finding`), so it lands in the report, obeys `policy`, and
sets the exit code, instead of the script inventing its own.

**It has no clock.** There is deliberately no `now` in the template language. A
document whose text changes on every run makes "is it stale" unanswerable, and
the answer would be red forever. A document that needs a date takes it from the
report — that is, from something that was **measured**.

### The template language

Small on purpose. The day it becomes a programming language, the shape of the
document has moved somewhere nobody reads again.

| | |
|---|---|
| `filter` `reject` | by a field being (or not being) one of the given values |
| `filterContains` `rejectContains` | by a field containing a fragment — which folder a path sits under is not an equality question |
| `filterLike` | same, with markdown emphasis stripped from both sides: an entry that ties a list to a heading must not die because somebody bolded a word |
| `groupBy` `sortBy` `pluck` `count` `get` | grouping is **sorted by key**, order inside a group is the report's; an order that shifts between runs would answer "is it stale" wrong every time |
| `before` `after` `lastBefore` `lastAfter` `split` `join` `replace` `trimPrefix` `trimSuffix` `trimSpace` `contains` `hasPrefix` `hasSuffix` `lower` `upper` `plain` `text` | text |
| `list` `append` `has` `add` `sub` `seq` | small helpers, for the counts and the sets a two-way check needs |
| `finding` | raises one, and renders nothing |

Missing keys are an **error**, not an empty string: a template that silently
writes `<no value>` says nothing on the day the report renames a field.

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

The generated text **and the binaries** are staged first, the gates named in
`gates` run **on the staged tree**, and only a green run reaches the release
directory — pages, checksums and `LATEST` cross over at one moment, because a
pointer announcing a version whose pages describe the one before it is a silently
broken publication. On red the staged text is kept and named in the message.

Each gate file brings its own sections: a `secrets` section measures the generated
text for leaks (a local path, a home directory, an address) and, with a second
file, for language; a `freeze` section holds each published document under its own
line cap. A page with no cap declared is not a page this repository publishes.

<!-- x3-dist version=v0.275.0 capabilities=a186d7b62d4c0d1683260322b5bc10fbb601b9da7bb6918567062dbf9145dd7c template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
