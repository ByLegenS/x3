# What the engine carries today

[The pages](INDEX.md) - [what x3 is](../README.md)

## The library, rule by rule

`x3 profile -library` prints this without reading any settings. The sets split
by what the caller has to **know**, not by taxonomy: one asks a repository for
nothing but its file extensions, two ask it to name where its parts are (one a
superset of the other — see [Two versions of one job](#two-versions-of-one-job)),
and one asks it to name the databases it develops and tests against. A
repository with no such facts does not call the set that needs them.

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

`go-parts@1` is not retired: a project that calls
it keeps exactly the five checks above, forever — a library rule is carried
inside the engine binary, so a project that named no version would have found
its rules changing under it on somebody else's upgrade. `go-parts@2` carries
the same five and adds the two that need a **manifest** — a fact `go-parts@1`
never asked a project to name. A project with no manifest file has no reason
to move; one that has both calls `go-parts@2` and drops nothing.

### `go-livedb@1` — three checks that need two databases and a prefix

| Rule | Lands in | What it measures | Asks for |
|---|---|---|---|
| `{tables}schema-columns-match-between-development-and-test` | `live.guards` | a column the development database carries that the migrations never produced in the test one | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |
| `{tables}schema-constraints-match-between-development-and-test` | `live.guards` | the same gap, for a constraint the application code already assumes is enforced | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |
| `{tables}schema-indexes-match-between-development-and-test` | `live.guards` | the same gap, for an index that changes a query's plan without changing its answer | `tables`, `dev-dsn`, `test-dsn`, `registry-table` |

Each check runs one query against `dev-dsn` and the same query against
`test-dsn`, and is red the moment the rows differ. `tables` is the same prefix
`go-parts` asks for; a project that calls both writes it once. `registry-table`
is excluded because it is the migration runner's own bookkeeping, not schema
a test run is meant to reproduce.

### A whole settings file

```json
{
  "profile": {
    "use": ["go-monorepo@1", "go-parts@1"],
    "with": {
      "text": ["go", "json", "md", "sql", "ps1"],
      "family": "services",
      "tables": "sand_",
      "migrations": ["db/migrations", "services/alpha/migrations"],
      "seal": "build/baselines/migration-seal.json",
      "builders": "build/release*.ps1"
    }
  },
  "arch": { "components": { "services": ["services/*/**"] } },
  "secrets": { "sources": ["**/*.go", "**/*.ps1", "**/*.md", "**/*.sql"] },
  "freeze": {
    "baselines": [
      { "name": "migration-seal",
        "sources": ["db/migrations/*.sql", "services/*/migrations/*.sql"],
        "seal": { "of": "content" }, "file": "build/baselines/migration-seal.json" }
    ]
  }
}
```

Measured on that tree: **22 lines of settings, 20 checks in force** — fifteen
rules, of which two list tokens build seven. Writing the same twenty checks into
the project's own sections is **245 lines** (both counted with one formatter).
The number that matters is not the difference but the slope: by hand every new
extension or migration directory costs another block, and in the call it costs
one list item.

Three rules ask for `secrets.sources`, `arch.components` and a seal baseline —
a call brings rules, not the facts a rule is measured against.

### What a carried rule may not be

This engine has one law that shapes the whole library: **a check that measured
nothing is not a green.** An empty subject set stops the run wherever it is
found. So a rule can only be carried if the set it calls has subjects by the
time it is called, and three candidates were dropped after being written and
measured:

| Candidate | Why it is not carried |
|---|---|
| every path a test opens is still on disk | a repository whose tests open no file by path has an empty set, and "there is no such test" would arrive as a red |
| a ceiling on the code that grows, frozen per file | the same, from the other side: a repository with no file near the threshold freezes nothing, so the threshold became a cap, which needs no baseline and no token |
| every build tag has its own vet step | both sides may legitimately be empty: no tag in the tree, no tag on the gate line |

A secrets pattern has the same shape for a different reason: an exemption that
excuses nothing is reported dead, so the address ranges a real repository
excuses cannot travel with the pattern. The carried password pattern therefore
ships with **no** exemptions, and a project adds its own with `override`.

<!-- x3-dist version=v0.175.0 capabilities=67925653b79a8165a912b94e81e1a9319d1f1981fe2afdcae635a3269b53ab60 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
