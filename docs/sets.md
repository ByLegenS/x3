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

**What enters the set is what was captured**, not the whole key, so it can be
compared with the constant that produced it. Because the extractor reads any text
file, this kind reaches past Go — a template calling names a script has to
define, and nobody compiles either.

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

A value naming nothing is `missing_target`. Values resolve against the
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

### `from: "x3"` — the engine's own roster

**Catches:** a rulebook sentence saying *"this one is guarded"* after the guard
was deleted, downgraded or never wired up. A `regex` over the settings file finds
the name in a rule turned down to `policy: "warn"` last month, which is exactly
the day the claim became false. `from: "x3"` reads the file **as a
configuration**, so the set carries the engine's verdict rather than the text.

```json
{ "left":  { "from": "regex", "file": "RULES.md", "select": "`x3: ([a-z-]+)`" },
  "right": { "from": "x3", "file": "x3.json", "select": "in-force" },
  "compare": "left-subset-of-right" }
```

`in-force` is every name the configuration puts in force — each rule, baseline,
pattern, guard and expectation by its `name`, each section by its key, and a
nameless mechanism by its **path in the configuration**. `commands` is the
subcommands it configures, and the map from a section to the command enforcing it
is the **engine's own roster** rather than a list kept beside it: a second copy
goes stale the day the engine grows a command, and a stale copy reports its own
blindness as *"all of it runs"*. Sections describing the engine's own workings
(`x3`, `update`, `baseline`, `cache`, `adoption`) enforce no check and are in
neither set. **`policy: "warn"` is not in force**, and neither is anything nested
under it.

**The file is read the way the engine reads it — parts included.** A large
project splits its configuration: the root declares its parts with `include`, and
every command merges them before measuring anything
([splitting the configuration](configuration.md#splitting-the-configuration)). The roster follows
the same reading. It did not always, and the bug is worth keeping written down: it
read the named file alone, so the day a project split its settings, every rule
that moved into a part left this set in silence. The rule went on running, and the
one question that could have noticed — *is this rule in force?* — answered **no**,
which makes the rulebook's true sentence about it red for a reason nobody can see.
A set that shrinks when a file is split is measuring the layout, not the rules.

### `invocations` — the commands a script actually runs

The other half of the same question: `commands` says what the settings put in
force, `invocations` says what the gate script really calls. Both answers come
from the engine, and that is the point — a hand-written pattern gets the shape of
a command name wrong in two measured ways. A name appearing **only inside a
string** is not a call, and a name carrying a colon (`guard:effective`) is one
word, not two. A pattern reading `x3 ([a-z]+)` counts the first and cuts the
second: the checker that never runs looks green, and the checker that does run
looks missing.

```json
{ "left":  { "from": "x3", "file": "x3.json", "select": "commands" },
  "right": { "from": "x3", "file": "gate.ps1", "select": "invocations" },
  "compare": "left-subset-of-right" }
```

Only names the engine actually carries enter the set. The engine is looked for by
its own name; a project calling it through a variable (`& $bin scan`) says so
with `invoke`, a list of patterns each carrying one capture group — the same
field, spelled the same way, that [`x3 adoption`](adoption.md#x3-adoption) reads.

<!-- x3-dist version=v0.122.0 capabilities=15ca9e0e91f95ecffad1e8ad49dbf97d72b4af46193928ec5bd51865cf8290e0 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
