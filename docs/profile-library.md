# What the engine carries today

[The pages](INDEX.md) - [what x3 is](../README.md)

## The library, rule by rule

`x3 profile -library` prints this without reading any settings. The sets split
by what the caller has to **know**: one asks for nothing but file extensions,
two ask where the parts are (one a superset of the other — see
[Two versions of one job](#two-versions-of-one-job)), one asks for the two
databases it develops and tests against.

### `go-monorepo@1` — nine checks, one list token

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `every-file-that-declares-a-function-is-named-by-a-test` | `arch.rules` | a file that declares something runnable, with no test in its directory naming it | — |
| `every-json-file-parses` | `syntax.checks` | a settings or data file that is read back as an empty one because it no longer parses | — |
| `the-module-file-carries-no-byte-order-mark` | `syntax.checks` | a mark in front of the module line, which every reader that resolves an import path has to get past | — |
| `no-merge-conflict-marker-survives-in-the-tree` | `syntax.checks` | a merge somebody stopped halfway, in the text the compiler says nothing about | — |
| `no-file-outside-a-test-imports-the-testing-package` | `syntax.checks` | the `-test.*` flags installed into a shipped binary's default flag set | — |
| `no-go-file-spells-a-path-of-one-machine` | `syntax.checks` | a path that compiles everywhere and reaches something on one disk | — |
| `a-password-written-next-to-its-name` | `secrets.patterns` | a password written beside the word that names it | — |
| `no-file-passes-a-thousand-lines` | `freeze.baselines` | a file past a thousand lines; a ceiling, so it takes no debt | — |
| `no-build-output-sits-in-the-tracked-tree` | `retire.groups` | a binary in the tree, whose source nobody can name; the count starts at zero | — |
| `every-{text}-file-decodes-as-utf8` | `syntax.checks` | bytes the gate and the editor read as different letters | `text`: a **list** of extensions |

### `go-monorepo@2` — the same nine, plus a status document's cap

Carries the nine above and adds `the-document-{docs}-stays-under-its-line-cap`
(`freeze.baselines`): a document past its declared line count, capped with no
debt. Asks for `docs` (a **list**) and a default `max`. `go-monorepo@1` is not
retired, for the reason `go-parts@1` is not.

Measured: `"use": ["go-monorepo@2"]`, `docs` naming two documents and
`"max": 120` is green at 118 and 190 lines while the second carries
`"override": { "the-document-docs/HANDOFF.md-stays-under-its-line-cap": { "cap":
{ "of": "lines", "max": 200 } } }`; grow the first to 121 and it is red under
its filled-in name; drop the override and the second goes red at 190 too.

### `go-parts@1` — five checks that need the project to name its parts

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `no-mechanism-body-lives-in-two-{family}` | `arch.rules` | the same body copied into two instances, so fixing one leaves the bug in the other | `family` |
| `no-{family}-instance-imports-another` | `arch.rules` | two things deployed apart, coupled where nobody looks | `family` |
| `a-table-is-spelled-only-by-the-{family}-instance-that-owns-it` | `arch.rules` | a table with two owners, neither knowing what the other assumed | `family`, `tables` |
| `every-published-migration-under-{migrations}-is-sealed` | `arch.rules` | a published migration whose text can still change after a database ran it | `migrations` (**list**), `seal` |
| `a-release-build-leaves-the-workspace-off` | `arch.rules` | a release binary linked against working copies, reporting a version nobody can build again | `builders` |

`family` is the name `arch.components` gives a set of sibling instances, and
also the directory they sit in. Declaring the component stays the project's
job: a component is a fact about one tree, and the library would be guessing.

### `go-parts@2` — the same five, plus two about a declared instance

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `a-declared-{family}-instance-carries-{legs}` | `arch.rules` | an instance whose manifest calls it complete, missing one of the directories that completeness promised | `family`, `manifest`, `legs` (**list**) |
| `a-declared-{family}-instance-carries-its-{entry}` | `arch.rules` | the same manifest, missing the entry script that would let anything run the instance | `family`, `manifest`, `entry` |

#### Two versions of one job

`go-parts@1` is not retired: a project that calls it keeps those five checks
forever — a library rule is carried inside the engine binary, so a project that
named no version would find its rules changing on somebody else's upgrade.
`go-parts@2` adds the two that need a **manifest**, a fact `go-parts@1` never
asked for: a project without one has no reason to move, one with both drops
nothing.

### `rulebook@1` — four checks that hold a document to what it says

A project's rulebook is read as instruction: the paths it names, the packages it
tells people to test, the gates it claims are enforced. Each of those is a promise
the tree can be measured against, and a stale one teaches the reader to distrust
the whole page.

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `paths-the-rulebook-names-still-point-somewhere` | `arch.rules` | a path the document names and the tree no longer carries | `rulebook`, `trees`, `least-paths` |
| `packages-the-rulebook-tests-still-exist` | `arch.rules` | a package the document tells people to test, which is not there | `rulebook`, `runner` |
| `engine-rules-the-rulebook-claims-are-really-in-force` | `arch.rules` | a gate the document claims and the settings do not run - believed, and false | `rulebook`, `claims`, `claim`, `settings`, `least-gates` |
| `root-holds-only-the-declared-directories` | `arch.rules` | a top-level directory nobody declared: a decision nobody made | `map`, `declares` |

The thresholds (`least-paths`, `least-gates`) are the check measuring **itself**:
a document that suddenly names far fewer paths than it used to is not a clean
document, it is a pattern reading the wrong lines - and without a floor that
failure passes as a green.

### `go-livedb@1` — three checks that need two databases and a prefix

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `{tables}schema-columns-match-between-development-and-test` | `live.guards` | a column the development database carries that the migrations never produced in the test one | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |
| `{tables}schema-constraints-match-between-development-and-test` | `live.guards` | the same gap, for a constraint the application code already assumes is enforced | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |
| `{tables}schema-indexes-match-between-development-and-test` | `live.guards` | the same gap, for an index that changes a query's plan without changing its answer | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |

Each check runs one query against `dev-dsn` and the same against `test-dsn`,
and is red the moment the rows differ. `tables` is the prefix `go-parts` asks
for, written once. `registry-table` is excluded: it is the migration runner's
own bookkeeping, not schema a test run reproduces.

### `go-livedb@2` — the same three, plus one rule for every other fact

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `{facts}-read-the-same-in-development-and-test` | `live.guards` | one check per fact the two databases must answer identically; the query lives **inside the item**, next to the name it is read for | `facts` (a list of items), `dev-dsn`, `test-dsn` |

The three checks above are the schema facts every repository shares. Everything
else a repository needs the two databases to agree on — a reference table's
rows, a set of enum labels — has the same shape and a different query, so it is
one rule measured once per item:

```json
"facts": [
  { "facts": "reference-rows", "reads": "SELECT code FROM ref_codes ORDER BY code" },
  { "facts": "enum-labels",    "reads": "SELECT unnest(enum_range(NULL::status))::text" } ]
```

`reads` is declared free text, so it may carry the spaces, quotes and
parentheses a query needs; it never reaches the check's name.

### A whole settings file

```json
{ "profile": { "use": ["go-monorepo@1", "go-parts@1"], "with": {
  "text": ["go", "json", "md", "sql", "ps1"], "family": "services",
  "tables": "sand_", "clone": 4,
  "migrations": ["db/migrations", "services/alpha/migrations"],
  "seal": "build/baselines/migration-seal.json",
  "builders": "build/release*.ps1" } } }
```

The same file also declares `arch.components`, `secrets.sources` and the seal
baseline the migration rule freezes against: a call brings rules, not the facts
a rule is measured against.

Measured on that tree (the gate's own sandbox, `check.ps1`): **23 lines of
settings, 20 checks in force** — fifteen rules, of which two list tokens build
seven. The same twenty checks written into the project's own sections are
**245 lines** (both counted with one formatter). What matters is the slope: by
hand every new extension or migration directory costs another block, in the
call one list item.

### What a carried rule may not be

One law shapes the whole library: **a check that measured nothing is not a
green.** A rule can only be carried if the set it calls has subjects by the time
it is called; three candidates were written, measured and dropped:

| Candidate | Why it is not carried |
|---|---|
| every path a test opens is still on disk | a repository whose tests open no file by path has an empty set, and "there is no such test" would arrive as a red |
| a ceiling on the code that grows, frozen per file | the same, from the other side: a repository with no file near the threshold freezes nothing, so the threshold became a cap, which needs no baseline and no token |
| every build tag has its own vet step | both sides may legitimately be empty: no tag in the tree, no tag on the gate line |

A secrets pattern has the same shape: an exemption that excuses nothing is
reported dead, so the ranges a real repository excuses cannot travel with the
pattern — the carried password pattern ships with **no** exemptions.

<!-- x3-dist version=v0.238.0 capabilities=31121a8487e25ce23e1f82e0cc6cd42327fdb42f00f845464ae9d2a1d459abfc template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
