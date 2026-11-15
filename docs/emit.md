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

**A data entry may be a pattern.** Definitions spread over many files — one view
per migration — are read into one document without a hand-kept copy of them:

```yaml
      data:
        views: db/migrations/*.sql   # every match, in path order
        note: docs/data/note.txt     # a plain path is one file, as before
```

`.Data.views.Text` and `.Data.views.Lines` are every matched file one after the
other; `.Data.views.Files` lists them apart, each with `.Path` (root-relative),
`.Text` and `.Lines`, so a template can say which file a line came from. A plain
path reads exactly as it did (its `Files` holds that one file). The wildcard sits
in the **file name** only: `db/*/views.sql` is refused when the settings are read
(exit 2), because a new file under a wildcard directory would not reopen a cached
step. A pattern that matches no file is refused too — an empty set reads like a
source with nothing to say. The pattern's directory is an input of the run: a
migration added later reopens a cached `emit -check` step and turns it red.
Control experiment: `emit data pattern control experiment` in `x3.yaml` (fresh
green, a migration added red, regenerated green, empty set and wildcard directory
refused). Measured against the engine before: the same entry exits 2 with
`cannot read data "views": open db\migrations\*.sql`. Cache arm, one-step gate on
a copy: second run `cache 1 hit`; a migration added → miss, `stale_document`;
`x3 emit` → miss, green; next run a hit.

**`-check` walks the tree once and recalls the box criteria.** A check writes
nothing, so the walk is kept for the whole run and the box list a document prints
is measured with the record `x3 boxes` keeps (see `x3 boxes`, a criterion recalled
by its spelling and its files' bytes). A run that writes does not keep the walk:
the page it writes is a new name a later criterion would have to see. Measured on
the pilot (2026-11-15, output byte-identical): 16.9 s and 276 s of processor
time → 8.1 s and 27 s; the parallel criteria had each walked the whole tree at the
same moment. No Go package loading was on the path: the profile held none.

### The three things a hand-written generator is missing

**It cannot go stale.** `-check` writes nothing and asks whether the file on
disk is what the template produces today; a difference is `stale_document`, and
it is a `block` by default. A generated document nobody regenerates is worse
than a missing one: it is read as current.

The compared copy is an input of the run like the template and the sources: a
gate step that runs `emit -check` is opened again when somebody edits the
generated file by hand, without `-force`, and a page that does not exist yet
ties the step to its directory. Before this the step was keyed on the template
and the sources only, and a hand-broken page came back green from the cache
(`cache 1 hit, 0 red`). Control experiment, run on a copy of
`internal/emit/testdata/pages` with a one-step gate: edit `out/INDEX.md` → the
step is red (`stale_document`); `x3 emit` → the next run is a cache hit, green;
the engine before the change → the edited page is a cache hit, green.

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
| `sortByCount <list-field> <list>` | records by the **length** of a list field, most first, equal ones in the report's order — `sortBy` compares strings and puts `10` before `9` |
| `before` `after` `lastBefore` `lastAfter` `split` `join` `replace` `trimPrefix` `trimSuffix` `trimSpace` `contains` `hasPrefix` `hasSuffix` `lower` `upper` `plain` `text` | text |
| `list` `append` `has` `add` `sub` `seq` | small helpers, for the counts and the sets a two-way check needs |
| `finding` | raises one, and renders nothing |

Missing keys are an **error**, not an empty string: a template that silently
writes `<no value>` says nothing on the day the report renames a field.

### A page per item, and a part two templates share

A list whose every item deserves its own page is one document with `each`: the
dotted path of a list in the report (`.` is the report itself), and an `out`
whose **file name** carries the field that names the page. The template sees
the item as `.Item` and its own path as `.Out`. `parts` are template files read
into **every** template of the section, so a block written once with `define`
is called from two templates with `template` — and changes in both on the same
run.

```yaml
emit:
  parts:
    - docs/templates/fields.tmpl        # {{define "fields"}}...{{end}}
  documents:
    - name: index
      out: docs/items/INDEX.md
      template: docs/templates/index.md.tmpl
      from: docs/data/items.yaml
    - name: items
      each: items                       # report key holding the list
      out: docs/items/{id}.md           # one page per item, named by its id
      template: docs/templates/item.md.tmpl
      from: docs/data/items.yaml
```

`-check` measures **every page on its own**, by name: a page that differs is
`stale_document`, a page the list produces and the disk lacks is
`stale_document` marked `ABSENT`, and a file in the set's directory that matches
the pattern but no item produces any more is `orphan_document` — the page of an
item that left the list is still read as current until somebody deletes it. A
writing run deletes it (`PRUNED`); another document's declared `out` in the
same directory (the index above) is never an orphan.

Hand-written pages may share the set's directory when the set names them:

```yaml
    - name: items
      each: items
      out: docs/items/{id}.md
      keep: [guide-*.md, about.md]      # file-name patterns in docs/items/
```

A file matching `keep` is neither `orphan_document` under `-check` nor pruned by
a writing run. A `keep` pattern that matches no file the list does not produce
is `dead_keep` — **always a block**, not open to `policy`: a keep with nothing
left to keep would shelter the next real orphan without a word. `keep` without
`each` is refused (only a page set prunes), and so is a pattern holding a `/`.
Control experiment: `emit keep control experiment` in `x3.yaml`.

Refused before anything is compared, exit 2: two items naming one page, a name
that is not a file name (a separator, `..`, a character a file system refuses),
`each` without a placeholder or a placeholder without `each`, a placeholder in
a directory, two page sets in one directory, two parts defining one name, and a
template defining a name a part already defines — `text/template` would keep
whichever was read last, and "the part changes in both templates" would quietly
stop being true for one of them.

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

The tables `codes` and `results` name (variables of package `main`) do not ride
the command line: they are written into a Go file that `go build -overlay` places
in the package directory, so the tree stays untouched. With one row per entry
point the table reached 59 KB, and a Windows command line stops at 32 KB — the
build never started (`The filename or extension is too long`). A variable of any
other package still goes with `-X`.

`SHA256SUMS.txt` is written from what was actually built. `LATEST` is written
**only** for a clean tagged tree: a build from `v1.2.3-2-gabc123` is a test build,
and a pointer naming a version nobody can check out is worse than no pointer.

A release run **by a binary that sits in its own release tree** is refused before
anything is built: a running binary cannot be overwritten, and learning that at
write time leaves half a tree. The file is compared by identity, not by spelling:

```
x3 release: this release runs from out\tool.exe, a file it would write; a running binary cannot be overwritten, so nothing is built or written - run a copy of it from outside out and release again
```

Measured: v0.338.0 was run from its own output path; the Linux binary was written,
the Windows one could not be, and `SHA256SUMS.txt` never was.

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
A gate file whose section does not load (an unknown field, a broken pattern)
stops the release with exit 2 and nothing written: a gate that cannot be read has
not passed. Measured: such a file was skipped and a leaking page went out green.

The gates above measure the **generated** text, not the source. A source whose own
gate is red is stopped by `release.gate`: the gate steps it names (the patterns
`gate -only` takes) run on the source tree before anything is built, and a red or
unreadable step ends the run with nothing built, written or tagged. Measured: two
releases went out with the source's architecture and syntax steps red, because
nothing in the chain asked the source's gate. A third went out with a red inline
example, because the list named only the static steps; this repository names its
examples step too. Examples of unchanged packages are recalled from the record, so
on a warm tree the step costs about a minute (measured 53.5 s).

```yaml
release:
  gate: [architecture, syntax, inline examples]   # gate steps that must be green before a build
```

```
x3 release: the gate steps the release section names (the red step) are not green (gate exit 1); nothing is built, written or tagged
```

A release writes nothing behind its own tag. A new page within its family's
allowance is green either way; a run that publishes a release version **holds**
it — measured, named, not written — and a test build (`-allow-dirty`) records it:

```
HELD page-follows-its-source publish/docs/fresh.md: a new page within the family's allowance of 5; it counts 5 and from here it only shrinks; not written - this run publishes a release, and a record written now lands behind its tag (a test build, -allow-dirty, writes it)
```

Measured on this repository: three releases that each added a page wrote the
tracked line-cap baseline after the tag, and the commit that carried it left the
tag behind its tree until the next release.

### The release tree is tagged where it lives

When the release tree is a repository named under `published` with its remote,
a release run commits what it wrote, tags that commit with the version, pushes
the branch and the tag, and then measures the tag with the very questions
`x3 published` asks — its own sentence is not the proof:

```
x3 release: ../publication carries v1.2.0 at the release commit 4fcde71c1a2b (committed) and origin has it
```

A tag that already names another commit is not moved, a push that cannot reach
its remote is red, and a tree not named under `published` is written and left
untagged, with a sentence that says so. A test build (`-allow-dirty`) tags
nothing. Measured: the v0.337.0 chain pushed the public repository without its
tag, and `published` read `tag_never_created` until a hand made it.

Only what this run wrote is committed: a change in the release tree it did not
write - a file left by hand or by an earlier run - stops the landing, named,
before any commit, because no gate has read it:

```
x3 release: the release tree ../publication holds 1 change(s) this run did not write and no gate read: notes.md; nothing is committed, tagged or pushed
```

The `secrets` section names the pages by a pattern (`docs/*.md`), not one by
one: a page written later is read the day it is born. Measured on this
repository: a hand-kept list read 109 of 130 published pages and was green while
one of the other 21 carried a consumer's variable name; with the pattern the same
tree is red at 7 lines.

<!-- x3-dist version=v0.380.0 capabilities=451c7c6ab59714f7ae80efc82ab6029b3a61b21aa0e436344a7fb54d479d80af template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
