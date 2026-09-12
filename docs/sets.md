# The two sets a rule compares

[The pages](INDEX.md) - [what x3 is](../README.md)

## Two sets, and how they must agree

A `consistency` rule reads two sets out of the tree and holds them to each other.
The extractors below are the same ones [`x3 freeze`](freeze.md#x3-freeze) uses for a
baseline's `set`, so what is written here about a hatch or a filter holds there
too.

### `consistency` — two sets that must agree

**Catches:** two sets drifting — a set of codes produced in code and a dictionary
giving each of them a message, so the user reads a raw key on screen.

```json
{ "kind": "consistency", "sources": ["internal/**/*.go"],
  "left":  { "from": "go",   "select": "const-set:Code" },
  "right": { "from": "json", "file": "i18n/en.json", "select": "keys:error.*" },
  "compare": "left-subset-of-right" }
```

| `from` | Reads | `select` |
|---|---|---|
| `go` | string constants of a named type | `const-set:<Type>` |
| `json` | the keys of one file, nested keys flattened to `a.b.c` | `keys:<pattern>` |
| `regex` | one capture group, read **line by line** | the pattern itself |
| `x3` | a settings file read as a **configuration**, or a script read as **calls** | `in-force`, `commands` or `invocations` |
| `tree` | the paths themselves — which directories and which files exist | `dirs:<pattern>` or `files:<pattern>` |

**What enters the set is what was captured**, not the whole key, so it can be
compared with the constant that produced it. Because the extractor reads any text
file, this kind reaches past Go — a template calling names a script has to
define, and nobody compiles either.

**An emptied declaration is a deleted one.** `keys:` records a key whatever its
value holds, so `"tables": []` reads exactly like `"tables": ["invoices"]` while
deleting the field is caught. The gate could then be closed by emptying the
declaration it measures. `"empty": "absent"` reads a key whose value is `[]`,
`{}`, `""` or `null` as the missing field it effectively is:

```json
"left": { "from": "json", "file": "manifest.json",
          "select": "keys:*.tables", "empty": "absent" }
```

A number or a boolean is **never** empty — `0` and `false` are answers, not the
absence of one. The default is `"counts"`, today's reading, written down so it
cannot change by itself, and the field belongs to `json`: only a container can
be empty.

This is a **reading, not an escape hatch**, and it has no dead law. It cuts both
ways — an emptied key leaves the set, so a name still standing on the other side
turns red, while a name empty on both sides falls silent together. Declaring it
on a tree with nothing empty is not a dead declaration but a clean one: today it
drops nothing, tomorrow it will.

**A value written in pieces.** A path assembled by the language —
`os.path.join(ROOT, "a", "b")` — has punctuation between its pieces, and a single
capture takes the whole block. `parts` reads the pieces **inside** what `select`
captured; `join` puts them back (`"a", "b"` → `a/b`), while `each` says the block
carries **many** values, one per piece. A line reading
`go test ./web/site/ ./internal/core/` names two packages, and joined they become
a path missing on every run:

```json
"left": { "from": "regex", "select": "go test((?:\\s+\\./[A-Za-z0-9_./-]+)+)",
          "parts": "\\./([A-Za-z0-9_/-]+?)/?(?:\\s|$)", "each": true }
```

Exactly one of `join` and `each` is written. RE2 keeps only the **last** match of
a repeated group, which is why capture groups alone cannot do this.

**Where the value stands.** A value can be right and still be in the wrong
place. `region` carries the container a value sits in into the set
([The container a value sits in](sets-region.md#the-container-a-value-sits-in)),
and `holds` turns that container into a question — *was this call written
inside a condition that asks?* ([Is this call inside that
condition](sets-holds.md#is-this-call-inside-that-condition)).

**Prose is not code.** A name in a comment does not run, so `comments: "exempt"`
drops comment text before the pattern reads (`checked` is the default); `syntax`
declares per-extension markers, and `quoted[].line` reaches the comment of a
language embedded in a string — the `--` inside a raw SQL literal.

**A string is not always code either.** `strings: "exempt"` drops a match lying
**from its first character to its last** inside one string literal: a usage line
a script prints names a command without running it. The measure is where the
match *starts*, not whether quotes are involved — a call carrying its argument in
quotes (`-ArgumentList 'record'`) begins in code and still counts. Dropping every
quoted run instead would take that real call with it, and the count would lie in
the other direction.

**One comparison per component.** By default both sets are read out of **every**
source the rule reaches and merged into one pool. That answers *"is this value
declared somewhere in the project"* — which is a different question from *"does
each application declare what it uses"*, and the merged pool **cannot** ask the
second one: a package one app imports without declaring is declared by another
app, the difference dissolves in the pool, and the rule reports a green it did
not earn. `per` names a component and asks the comparison once per instance,
with only that instance's files on each side:

```json
{ "kind": "consistency", "per": "apps",
  "sources": ["apps/**"],
  "left":  { "from": "regex", "select": "\"[a-z.]+/(core/[a-z]+)\"", "sources": ["apps/*/**.go"] },
  "right": { "from": "regex", "select": "uses (core/[a-z]+)", "sources": ["apps/*/uses.txt"] } }
```

Every finding names its instance (`object`), so the same value drifting in two
places is two findings and each can be baselined on its own. The component needs
a star in its patterns — a component with one instance would make this a copy of
the merged reading. A side that reads one fixed document (`json`, `x3`, or any
`file`) is refused for the same reason: a text that cannot vary by instance is
not being compared per instance.

An **empty instance is not red**: an application that uses no core package and
declares none agrees. Blindness is measured over the rule, not the instance — a
side no instance can read is `empty_scope`. Asked per instance instead, every
correctly silent application would shout; not asked at all, an unreadable side
would pass as agreement.

### `left-exists-on-disk` — does the path still point at something?

**Catches:** a gate carrying a path constant that keeps working after the path
moves — it finds nothing, reports nothing and **exits `0`**. The most expensive
form is a criterion phrased as an absence: once the root is gone it is true
forever, and work that was never done reads as finished.

Not [`containment`](arch.md#containment---a-components-parts-stay-under-its-root): there
the question is where an existing file belongs, here whether the thing pointed at
exists at all.

```json
{ "kind": "consistency", "sources": ["scripts/gates/**/*.py"],
  "left": { "from": "regex", "select": "\"((?:internal|cmd|docs)/[A-Za-z0-9_./-]+)\"" },
  "compare": "left-exists-on-disk",
  "absent": { "internal/legacy/importer": "deleted in the migration" } }
```

A value naming nothing is `missing_target`, **reported against the file that
carries the value** — a finding has to open somewhere, and one that names only a
value leaves a baseline record nobody can review a year later. A value written
in several files is reported against the first of them, so the same tree gives
the same bytes. The one finding with no file is an `absent` entry **nothing
names any more**: there is no file to point at, which is what the finding says.
Values resolve against the
**repository root** unless `relativeTo: "source"` resolves each against the
directory of the file carrying it, which is what a test reading `"../../x.go"`
needs; the same text in two files is **two targets**. `absent` is a
**path → reason** map, and an exemption with no reason is refused.

| Hatch | What it takes out | When it goes stale |
|---|---|---|
| `exclude` | the **file** that would have been read | `dead_exclusion` |
| `skip` | a **value**, before any verdict | `dead_filter` |
| `absent` | the **verdict** on a measured value | `dead_exemption` |

`dead_exemption` is raised both when the excused path is on disk again and when
it is named nowhere any more: an exemption list that only grows is a gate
carrying its own silencer.

⚠️ **A path whose presence depends on the machine belongs in `skip`, not
`absent`.** A build output, a credentials file, or the sand a control experiment
writes on purpose is there on one checkout and gone on the next; `absent` states
a fact about the disk, so it turns `dead_exemption` the first time the file
appears, and dropping it turns `missing_target` the first time it does not.
`skip` takes the value out before any verdict, so neither state is judged, and
it still cannot rot in silence — a pattern that sifts nothing is `dead_filter`.
One pattern may cover a whole family, which is what a project that keeps its
control-experiment fixtures inside its gate scripts needs.

<!-- x3-dist version=v0.140.0 capabilities=b8445c1227f160e93100c6d30aa776467b63ccee66c2b1d4455c262b73b1ed73 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
