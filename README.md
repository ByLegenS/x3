# x3 — one engine, every audit

x3 reads a Go code base and the live environment it is about to run against,
and answers one question before the work starts: **is anything not what this
code assumes?** It checks contracts written as ordinary comments (`//x3:` directives,
so the compiler never sees them), it checks that the source is written in one
language outside its comments, it checks the shape of the project against the
architecture the project itself declares, and it checks the running world — a
database row, an HTTP endpoint, a command's output — and refuses to launch when
the answer is wrong.

**This repository ships binaries only.** The source is private. Everything the
published binaries can do is documented here: this file says what x3 **is**, and
every capability has a page of its own under [docs/](docs/INDEX.md) that says how
it is **used** — the directive line, what it catches, a red and a green example,
and the settings it reads. All of it is generated from the engine's own capability
document at build time, so no page can describe a version that does not exist.

The lookup tables — field names, error codes, exit codes — sit beside the guides,
one reference page per family, listed in [REFERENCE.md](REFERENCE.md) and written
in the same run; every page links to the table it uses.

**Current version: `v0.158.0`**

## Download

| File | Platform | Size | SHA256 |
|---|---|---|---|
| `x3-windows-amd64.exe` | windows/amd64 | 15.4 MB | `57bcd4497db14585362e87773482cdec3b2b2ef351315b9f51ba8de8540c75f6` |
| `x3-linux-amd64` | linux/amd64 | 15.1 MB | `382659a218d962f4257ba5ccaf8e9eee360c4ebddc4f5881c5a953e485a03885` |

Both binaries are static (`CGO_ENABLED=0`) and carry no runtime dependency.

### Verify what you downloaded

The checksums above are also in `SHA256SUMS.txt`, next to the binaries:

```
sha256sum -c SHA256SUMS.txt                                  # Linux
Get-FileHash .\x3-windows-amd64.exe -Algorithm SHA256        # Windows
```

The build is reproducible — `-trimpath -buildvcs=false -ldflags "-s -w
-buildid= -X main.version=<tag>"` with `CGO_ENABLED=0` — and every release is
built twice and published only when both passes produce the same hash. A
checksum that does not match the table is not the binary that was published.

### Install

Rename the file to `x3` (`x3.exe` on Windows) and put it on your `PATH`, or
call it by path from a gate script. There is no installer and nothing is
written outside the file you downloaded.

```
x3 version
```

prints the embedded release tag — the same tag as the download you took. A
binary built outside a release prints `unreleased`.

From then on the binary updates itself:

```
x3 update
```

It reads the newest published tag, downloads the binary for this platform,
verifies its SHA256 against the published list and only then replaces the file
it is running from. A sum that does not match is refused and nothing is
touched. `x3 update -check` answers the same question without installing
anything, and a project can require a minimum version in its own `x3.json` so
that an old binary refuses to run at all.

### Pin a version

A gate should pin a tag and a checksum, not "the latest file". Every release is
tagged in this repository, so a fixed URL fetches a fixed binary:

```
https://raw.githubusercontent.com/ByLegenS/x3/<tag>/x3-linux-amd64
https://raw.githubusercontent.com/ByLegenS/x3/<tag>/x3-windows-amd64.exe
```

Fetch it, compare the SHA256 against the one you pinned, and treat a mismatch
or a missing binary as **red** — not as a skipped step. A gate that quietly
passes because its tool was missing is worse than no gate at all.

## First run

```
x3 scan  ./internal/...         # directives in the source
x3 lang  -config x3.json .      # one language outside comments
x3 arch  -config x3.json .      # which component may import which
x3 freeze -config x3.json .     # frozen lists that only shrink
x3 surface -config x3.json .    # the exported API, which may only grow
x3 docs  -config x3.json .      # changes that must not travel alone
x3 secrets -config x3.json .    # credentials that got into the source
x3 boxes -config x3.json .      # open work, in a file or in the documents
x3 syntax -config x3.json .     # files no compiler reads, parsed anyway
x3 scope  -config x3.json .     # a change that must stay in its lane
x3 placement -config x3.json    # a rule declared where the region it names can read it
x3 record -listen :9100 -target http://localhost:8080 -ledger api.jsonl  # traffic, written down
x3 replay -target http://localhost:8080 -ledger api.jsonl          # and compared with it
x3 outbound serve -listen :9101 -ledger out.jsonl                  # the far side, from the ledger
x3 guard -config x3.json -- go test ./...   # live checks, then the command
x3 guard:effective -config x3.json          # the setting on paper vs in force
x3 testdb run -config x3.json -- go test ./...   # a fresh database for this run
x3 adoption -config x3.json .   # how much of this engine the project actually runs
x3 published -config x3.json     # the release the pointer announces, tagged and pushed
```

Exit codes are the same for every command: **0** green, **1** red, **2** usage
or I/O error. When `guard` launches the command, the command's own exit code is
returned instead.

Project configuration lives in one file, `x3.json`: the `language` section for
the language gate, the `arch` section for the architecture rules, the `freeze`
section for the frozen baselines, the `surface` section for the exported API, the `docs` section for coupled changes,
the `secrets` section for the leak scan,
the `boxes` section for the open-work list, the `record` section for
what a recording must hide, the `replay` section for what may differ,
the `cache` section for where a run may remember what it measured, the `live`
section for the guards, the `effective` section for the recorded-versus-in-force
comparisons, the `testdb` section for run-lifetime databases, the `adoption` section for
which of these the project is actually running, the `published` section for the
tags a release must carry. A large repository
splits that file: the root declares its parts with `include`, lists are added
and objects merged, and anything else set twice stops the run. All of them are documented below, with the schema and a worked
example.

---

**What the engine does today** — every capability, what it catches, and one
worked example, quoted from the control samples in the repository. Each capability
has a page of its own, listed below. If something is on the roadmap and not on one
of these pages, it does not exist yet.

> **Documentation gate.** A change under `internal/` or `cmd/` must carry a
> change under `docs/` in the same diff, or `check.ps1` turns red. See
> [The documentation gate](docs/experiments.md#the-documentation-gate).

## Guides

Every capability has a page of its own; the table is generated from the same
markers that split this document, so a page cannot be missing from it.

| Page | What it covers |
|---|---|
| [Directives in the source](docs/scan.md) | the `//x3:` dictionary, the scopes a directive may sit in, the report and its expectations |
| [Where a directive may sit](docs/layout.md) | the placement the Go formatter writes, measured by the engine itself, so that a formatting run cannot move a directive behind your back |
| [Where a pattern binds](docs/patterns.md) | how `^` and `$` are read against a file, and where a line ends |
| [Inline examples that run](docs/case.md) | an example that calls the declaration it sits on, with a state given to it and an aspect of the result asserted |
| [The names an example may reach](docs/case-imports.md) | a package no source file can import, a package its path cannot spell, and the two places a declaration may be written |
| [The type an example declares](docs/case-types.md) | a fake with methods, written in a comment and alive only inside the generated test, next to the declaration that serves nothing |
| [Examples behind a build tag](docs/case-tags.md) | the tag a package must be built with, the run that carries it, and the examples a run refuses to pass over in silence |
| [What a run says](docs/case-findings.md) | the finding codes, propositions that stop at the first failure, the settings and the report |
| [One language outside comments](docs/lang.md) | the dictionary run in reverse: the allowed language, and every token outside it |
| [The words a language reserves](docs/lang-keywords.md) | the keywords of the language being read, which no dictionary carries and which arrive as a finding at every occurrence |
| [The three scopes of a language run](docs/lang-scope.md) | which files are read, whether their strings are read at all, and the names of files and directories that no file's content ever carries |
| [The field name written inside a string](docs/lang-fields.md) | the keys of a structured log line, read as identifiers even where a project has declared its strings free |
| [The shape of the project](docs/arch.md) | the import graph and nine further rule kinds, against the components a project declares |
| [Does anybody touch this file](docs/arch-pairing.md) | the counterpart a file must have, the two questions it can be asked, and which files are asked at all |
| [Which files the rule is about](docs/arch-pairing-subjects.md) | the property that says what a pairing rule measures, why it is neither an exemption nor an exclusion, and the elimination that is counted |
| [The example that stands in for a counterpart](docs/arch-pairing-example.md) | the inline example a migrated test leaves behind, what makes it count, and the price of declaring it |
| [The one line a rule may not reach](docs/arch-exemptions.md) | the reason a violation is excused with, the place that reason binds, and the exemption that stopped excusing anything |
| [Where a restricted value may appear](docs/arch-flow.md) | the places a handle may stand in, and why asking about it is not one of the escapes |
| [Prose is not code](docs/arch-prose.md) | the comment syntax that tells a rule's reading apart from the file's story, in every language and not only in Go |
| [The two sets a rule compares](docs/sets.md) | the consistency kind, the extractors that read each side, and the escape hatches they carry |
| [The tree read as a set](docs/sets-tree.md) | the paths themselves as one side of a comparison, so that a directory missing the very file a rule requires is still asked about |
| [The engine read as a set](docs/sets-roster.md) | the checks a settings file puts in force, the commands the binary carries, and the calls a gate script really makes |
| [The container a value sits in](docs/sets-region.md) | the section, block or card a value was written inside, carried into the set so it can be weighed against what the value itself says |
| [Is this call inside that condition](docs/sets-holds.md) | the region turned from a name into a question, so that a value counts only when the container it sits in holds what it should |
| [Where the container ends](docs/sets-closing.md) | the closing pattern that turns a boundary into a two-ended region, so that the values inside a block are read without the ones written below it |
| [Lists that may only shrink](docs/freeze.md) | a measured set or a number frozen to a baseline that growth turns red |
| [The exported API, which may only grow](docs/surface.md) | a removal or a changed signature is red, and the finding names who breaks |
| [Today's findings, frozen](docs/baseline.md) | adopting a gate on a tree that is not clean yet, without a thousand reds |
| [The sentence that says why a debt is held](docs/baseline-why.md) | the reason written next to a record by hand, carried through a file the engine regenerates, and what becomes of it when the debt is paid |
| [A baseline belongs to the root it measured](docs/baseline-root.md) | the coordinate system every recorded path lives in, and the narrower run that reads a mismatch as a debt paid |
| [A baseline two branches write](docs/baseline-parallel.md) | the split that keeps two regions out of one file, and the derived field a merge quietly gets wrong |
| [One debt, one file, one writer](docs/baseline-segments.md) | the split that gives a region or a rule its own baseline file, and the two axes a part may own |
| [Changes that must not travel alone](docs/docs.md) | a change under one path that requires a change under another in the same diff |
| [Credentials in the source](docs/secrets.md) | credential formats in any text file, masked in the report that names them |
| [The comment diet](docs/comments.md) | comment blocks over a limit, with the ratio to code kept as a warning |
| [Open work, measured](docs/boxes.md) | every box against the criteria that would prove it done, in both directions |
| [A work list written as a document](docs/boxes-document.md) | the checkboxes inside a note read as boxes, the states a project declares, and how a criterion is written beside an item - in prose or inside a table |
| [Three ways a criterion is lost in the writing](docs/criterion-writing.md) | the key that is not the criterion key, the sentence handed to a runner, and the place written as a line number |
| [Criteria that stopped measuring](docs/boxes-suspect.md) | the criterion that cannot fail, and the selector whose name has left the tree |
| [Before a name is removed](docs/holds.md) | which criteria hold a name that is about to be deleted, including the selector patterns a search cannot find |
| [Which hold is really a hold](docs/holds-weight.md) | the place a criterion looks at, weighed against the file being asked, and the record the engine refuses to decide |
| [A name that lives outside the list](docs/holds-elsewhere.md) | the gate scripts and workflow files a name is called from, declared because no engine can guess them |
| [What a gate's line holds](docs/holds-selectors.md) | the selector a gate hands its runner and the package its step runs, both read by declared patterns, next to the bare word that cannot be weighed |
| [Files no compiler reads](docs/syntax.md) | JSON, YAML, TOML, SQL and the rest, parsed anyway; a missing parser is red |
| [The file a shell decodes before it runs it](docs/syntax-encoding.md) | the bytes a parser reads its own way and the shell reads another, and the mark that settles it |
| [The parser that reads the name](docs/syntax-content.md) | the outside parser that decides what to read from the file name and passes a file it never checked, and the content handed to it instead |
| [The word another language owns](docs/syntax-ignore.md) | the lines a denied pattern catches but must not count, excluded in the grammar the credential scan already uses, and the exclusion that stopped excluding anything |
| [Which files the check is about](docs/syntax-subjects.md) | the subject set bound to what a file holds rather than to where it sits, and the elimination that is counted |
| [What a check does not read](docs/syntax-scope.md) | the file a source glob takes but the check must not weigh, and the comment that explains the rule rather than breaking it |
| [A change that stays in its lane](docs/scope.md) | a declared lane, and the change that enters it and also reaches outside |
| [Only the tests a change can reach](docs/test.md) | the unit graph, the cache, and what the measurement honestly shows |
| [One call per unit, one database per unit](docs/test-isolation.md) | what a shared process and a shared database hide, and what the isolation costs |
| [A skipped test is not a green one](docs/test-skipped.md) | counting what the runner skipped, and the policy that makes it red |
| [The test you forgot to write](docs/mutate.md) | the code broken on purpose, and the behavior no test noticed |
| [What a mutation run breaks, and what it says](docs/mutate-findings.md) | the operators, the text mutations, the fail-closed codes, and the findings a run names |
| [Traffic, written down](docs/record.md) | a run of the application recorded, redacted before it reaches the disk |
| [The recording, sent again](docs/replay.md) | compared field by field, with what is allowed to differ written down |
| [The live world, before the command](docs/guard.md) | `sql`, `http`, `exec` and multi-step trials, each one a warning or a block |
| [The reds a live gate starts with](docs/guard-baseline.md) | freezing the checks a running system fails today, so that a new failure is the only one that speaks |
| [Two schemas, one question](docs/guard-rows.md) | the row set a query answers with, weighed against the row set a second connection answers the same question with, and the difference named row by row |
| [The setting on paper against the setting in force](docs/effective.md) | a recorded value compared with the value the running system actually uses |
| [How much of this engine actually runs](docs/adoption.md) | the project measured against the engine's own command table |
| [How the engine is called](docs/adoption-invoke.md) | the spelling a project calls the engine by, the prose and the printed lines that are not calls, and where the line between them is drawn |
| [Binding the gate without a permanent red](docs/adoption-policy.md) | the policy object, the laws an exception carries, and the finding that audits the exceptions themselves |
| [The version, and how it updates itself](docs/update.md) | the embedded tag, the self-update, the pinned checksum and the minimum version gate |
| [The program a command name means](docs/commands.md) | a declared command resolved before it runs, so a failure names the program that actually ran rather than the name that was written |
| [A command started to measure cannot wait for a person](docs/unattended.md) | the console a measured child never inherits, the stdin it already reads as empty, and the streamed run this rule deliberately leaves alone |
| [A run leaves nothing behind](docs/leftover.md) | the processes a run started and did not take with it, counted from the container they cannot escape rather than guessed from their names |
| [A fresh database for this run](docs/testdb.md) | a template cloned per run, migrated, dropped, and the leftovers collected |
| [Getting the database ready](docs/testdb-setup.md) | the ordered steps a fresh database is prepared with, and the variables each one is handed |
| [Speed, the cache, and what a run leaves behind](docs/speed.md) | measured timings, the incremental cache, and the files the engine reads back |
| [One configuration, split across files](docs/configuration.md) | `include`, how lists and objects merge, and a real `x3.json` from a live project |
| [A rule is declared where it applies](docs/placement.md) | the region a rule's paths fall in, the settings file that has to hold it, and the rule that weighs two regions and belongs to neither |
| [Releases, and calling the engine from another project](docs/releases.md) | reproducible builds, and the gate script that pins a tag and a checksum |
| [Is the release really published](docs/published.md) | the tag a publication announces, measured in every repository and on every remote, and against the commit that carries the announcement |
| [Gaps we know about](docs/gaps.md) | what is not built, said plainly, next to what is |
| [Gaps in what a work list can say](docs/gaps-work.md) | the bounds of the open-work list, the examples that run beside it, and the one language the gate speaks |
| [Gaps in what a run reaches](docs/gaps-outside.md) | the bounds that begin where the source tree ends: the release it pulls, the toolchain it mutates through, the traffic it records, the live world it asks, and the database it borrows |
| [Control experiments, and the documentation gate](docs/experiments.md) | every capability proven able to go red, and the gate that keeps these pages current |

## What is built and what is not

Every capability below is implemented and has a control experiment in
`check.ps1` that proves it can go **red** — a green nobody has seen fail is not
evidence.

| Capability | What it catches |
|---|---|
| **Scanner** (`internal/scan`) | walks the Go AST, collects `//x3:` directives, resolves their scope |
| **Dictionary** (`internal/scan`) | six directive types; an unknown type or a malformed shape is red |
| **Inline examples** (`internal/cases`) | `x3 case` **calls** the declaration a `//x3:case` sits on; nothing is written into the tree |
| **Language gate** (`internal/lang`) | one language outside comments, against an embedded dictionary plus the project's `language.allow` |
| **Architecture rules** (`internal/arch`) | the import graph and nine further rule kinds, against the components a project declares |
| **Frozen baselines** (`internal/freeze`) | a measured set or number that may only shrink; `-update` records a shrink and refuses growth |
| **Frozen API surface** (`internal/surface`) | the exported Go API of declared packages, which may only grow; a removal or a changed signature is red, and the finding names who uses it |
| **Finding baseline** (`internal/baseline`) | today's findings frozen so a new gate can be adopted without a thousand reds |
| **Coupled changes** (`internal/docs`) | a change that must not travel alone; the exemption needs a written reason |
| **Secret scan** (`internal/secrets`) | credential formats in any text file, masked in the report |
| **Comment diet** (`internal/comments`) | comment blocks over a limit; the ratio to code only warns |
| **Open work** (`internal/boxes`) | each box in the work list measured against the criteria that would prove it done — both directions |
| **Syntax** (`internal/syntax`) | files no compiler reads, parsed anyway; a missing parser is red, not skipped |
| **Lane discipline** (`internal/scope`) | a change that enters a declared lane and also reaches outside it |
| **Selective tests** (`internal/test`) | only the units a change can reach, plus a cache for the run where nothing changed |
| **Recorded traffic** (`internal/record`) | a run of the application written down, redacted before the disk |
| **Replay** (`internal/record`) | the recording sent again and compared field by field |
| **Live guards** (`internal/live`) | `sql`, `http`, `exec` and multi-step trials, each `warn` or `block`, before a command launches |
| **Effective checks** (`internal/live`) | a setting as *recorded* against the same setting as it is *in force* |
| **Test databases** (`internal/testdb`) | a template database cloned per run, migrated, dropped, and the leftovers collected |
| **Incremental cache** (`internal/cache`) | keyed on engine version, configuration fingerprint and file content; off unless declared |
| **Settings placement** (`internal/placement`) | a rule whose paths all fall in one region, declared anywhere but that region's own settings file |
| **Adoption** (`internal/adoption`) | how much of this engine the project actually runs, measured against the engine's own command table |

What `x3 scan` itself implements is a **language check**, not a behavior check.
It answers three questions about every `//x3:` line: is the type known, is the
shape right, is the scope legal. It never calls your code. Running an example
*is* implemented, but as a separate gate — [`x3 case`](docs/case.md#x3-case). `x3 record`
and `x3 replay` do reach behavior, but only through the HTTP surface and only
over the traffic the recording saw. `x3 guard` reaches the outside world, but
checks the environment a run is about to happen in, not your code. A green
`x3 scan` means *"your directives are well formed"*, nothing more.

---

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
