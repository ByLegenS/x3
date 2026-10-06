# Who uses this file

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 uses`

**What it catches:** a file nothing in the tree names - a leftover page, an
example file no task reads, an image no page asks for, a data file the tool that
read it took with it.

```
x3 uses [-config <file>] [-out <file>] [-file <path>] [-cache <file>] [-no-cache] [-baseline <file>] [-update-baseline] [dir]
```

```
UNUSED data/stray.csv: unused
        no file, setting or link of this tree names it
x3 uses: 10 file(s) - 9 read - 8 weighed - 1 unused, 3 exempted
```

`-file <path>` turns the question around and prints every user of one file with
the line that names it (exit 1 when nobody does):

```
USED   lib/page.html <- lib/lib.go:0 page.html
```

**What counts as a use.** A mention is resolved against the run's own file list,
so a word that names no file costs nothing.

| Source | A use | Not a use |
|---|---|---|
| Go | an import of a package of the tree's modules (every `.go` file of that directory), a `//go:embed` file or one-directory glob, a path in a string | a path in a comment |
| Settings (`.yaml`, `.json`) | an exact path in any value; a glob under a read key (`sources`, `include`, `run`, `files`, `inputs`, `args`, `command`) | a glob with `**`, a bare `*` or no wildcard (a scope); anything under `exclude`, `ignore`, `skip`, `keep` |
| Web (`html`, `css`, `js`) | `href`, `src`, `url(...)`, `import`, a path in a string; a template `logo-${t}.png` as a glob | `${dir}/${name}.html` (it could be any page); a comment |
| Documents (`.md`) | a link; a path to another document in backticks | a backticked data file or a bare name: prose describes, it does not depend |

**A scope is not a use.** `apps/**`, `assets/*`, a directory embedded whole: such a
pattern holds whatever is put under it, used or not, so the files it matches
still need a user of their own - the page inside an embedded directory is asked
for by the links of the other pages.

A mention is read from the writer's directory upward (a page asks `assets/x.png`
of its site root, a program asks `data/a.json` of the working directory). A bare
name joined in code (`filepath.Join(dir, "a.json")`, a route `/about` served by
`about.html`) is taken when exactly ONE file of the tree ends that way; two
candidates are no answer.

Three shapes a path is built in without being spelled:

| Written | Names |
|---|---|
| a document or link naming a directory: `docs/notes/` | its README, the page a reader opening it is shown (a bare `//` names nothing) |
| a quoted directory and a name list a line or two above it: `for (const n of ['login', 'lock'])` / `part('/auth', n)` | `auth/login.*`, `auth/lock.*` (four lines up is another statement) |
| a backticked directory and a bare document name on ONE line: `` `gates/` (see `OKU.md`) `` | `gates/OKU.md` (the name on the next line is not joined) |

What still needs a declaration: a file read by listing its directory (a scenario
folder a tool walks), a name built from a value the text never holds. The tree
cannot show such a use; the project says it, with its reason, under `uses.exempt`.

**What needs no user.** A main package, a test file, `go.mod` / `go.sum` /
`go.work`, `.gitignore` / `.gitattributes`, every file of the configuration's own
name, and the engine's own work lists. The engine's records (the baseline
directory, the box archive) are blind: they name paths because they remember
them. Everything else the project declares, each with a reason:

```yaml
uses:
  exempt:                      # needs no user
    - path: LICENSE
      why: read by whoever receives the binary
  blind:                       # its mentions are not uses
    - path: docs/CHANGELOG.md
      why: a history names what is gone
```

An exemption with no `why` is red (`exemption_without_why`), and one that matches
no file is red too (`dead_exemption`): it would hide the next file it catches.

**Why "at least one user" and not reachability from the entry points.** Two dead
files that name each other survive this question; reachability would catch them,
but every document that only a person opens would turn red with them. One user is
the question the tree can answer without guessing.

**The cache.** What a file mentions is a function of its bytes: it is remembered
per file (salted by the `uses` section alone), and only the join runs again. A
second run on an unchanged tree extracts nothing; a changed file is read again
(`cache 8 hit(s), 1 miss(es)`). The remembered value is one text per file, not a
tree of fields, and the configuration is merged once per run: on a tree of 1 932
files the run takes 0.65 s unchanged, 0.67 s with one file changed, 0.75 s with
`-no-cache`.

Findings carry a baseline like every other audit: the finding's name is its
path, and `-update-baseline` only shrinks.

<!-- x3-dist version=v0.334.0 capabilities=05d32e130d9cedb16eec9c10f66b50c62f766394fce85acb384d8848e14d5f6d template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
