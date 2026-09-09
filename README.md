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
published binaries can do is documented below, and this file is generated from
the engine's own capability document at build time, so it can never describe a
version that does not exist.

**Current version: `v0.56.0`**

## Download

| File | Platform | Size | SHA256 |
|---|---|---|---|
| `x3-windows-amd64.exe` | windows/amd64 | 14.1 MB | `25d0dc9078e61460b9acc2ba048f4ab7623cdc503ba0b41feea9d245a87e0235` |
| `x3-linux-amd64` | linux/amd64 | 13.7 MB | `838f6a5d8d6e23218447256c7d3aaeceb26e9484f7daec921fa670cc93b47e51` |

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
x3 record -listen :9100 -target http://localhost:8080 -ledger api.jsonl  # traffic, written down
x3 replay -target http://localhost:8080 -ledger api.jsonl          # and compared with it
x3 outbound serve -listen :9101 -ledger out.jsonl                  # the far side, from the ledger
x3 guard -config x3.json -- go test ./...   # live checks, then the command
x3 guard:effective -config x3.json          # the setting on paper vs in force
x3 testdb run -config x3.json -- go test ./...   # a fresh database for this run
x3 adoption -config x3.json .   # how much of this engine the project actually runs
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
which of these the project is actually running. A large repository
splits that file: the root declares its parts with `include`, lists are added
and objects merged, and anything else set twice stops the run. All of them are documented below, with the schema and a worked
example.

---

**What the engine does today** — every capability, what it catches, and one
worked example, quoted from the control samples in the repository. If something
is on the README roadmap and not in this file, it does not exist yet.

> **Documentation gate.** A change under `internal/` or `cmd/` must carry a
> change under `docs/` in the same diff, or `check.ps1` turns red. See
> [The documentation gate](#the-documentation-gate) at the bottom.

## Contents

- [What is built and what is not](#what-is-built-and-what-is-not)
- [`x3 scan`](#x3-scan)
- [Scopes](#scopes)
- [How a pattern is read](#how-a-pattern-is-read)
- [The dictionary](#the-dictionary)
- [Error codes](#error-codes)
- [The JSON report](#the-json-report)
- [Expectations](#expectations)
- [`x3 case`](#x3-case)
- [`x3 lang`](#x3-lang)
- [`x3 arch`](#x3-arch)
- [`left-exists-on-disk`](#left-exists-on-disk--does-the-path-still-point-at-something)
- [`from: "x3"`](#from-x3--the-engines-own-roster)
- [Scope integrity](#scope-integrity)
- [`x3 freeze`](#x3-freeze)
- [The count mode](#the-count-mode)
- [A cap with no baseline](#a-cap-with-no-baseline)
- [`x3 surface`](#x3-surface)
- [The direction is the whole point, and it is `freeze` inverted](#the-direction-is-the-whole-point-and-it-is-freeze-inverted)
- [Who breaks](#who-breaks)
- [The finding baseline](#the-finding-baseline)
- [`x3 docs`](#x3-docs)
- [`x3 secrets`](#x3-secrets)
- [Excluding what the pattern also catches](#excluding-what-the-pattern-also-catches)
- [`x3 comments`](#x3-comments)
- [`x3 boxes`](#x3-boxes)
- [A list that is finished](#a-list-that-is-finished-and-where-it-goes-next)
- [`x3 syntax`](#x3-syntax)
- [`x3 scope`](#x3-scope)
- [`x3 test`](#x3-test)
- [A test's import does not travel](#a-tests-import-does-not-travel)
- [`x3 record`](#x3-record)
- [`x3 replay`](#x3-replay)
- [`x3 guard`](#x3-guard)
- [Choosing which guards run](#choosing-which-guards-run)
- [`x3 version`](#x3-version)
- [`x3 update`](#x3-update)
- [`update.pin`](#updatepin--the-checksum-the-project-itself-vouches-for)
- [Live guards in `x3.json`](#live-guards-in-x3json)
- [`kind: "steps"`](#kind-steps--a-trial-not-a-reading)
- [The guard report](#the-guard-report)
- [`x3 guard:effective`](#x3-guardeffective)
- [Effective checks in `x3.json`](#effective-checks-in-x3json)
- [The effective report](#the-effective-report)
- [`x3 testdb`](#x3-testdb)
- [`x3 adoption`](#x3-adoption)
- [How the engine is called](#how-the-engine-is-called)
- [Speed](#speed)
- [Files the engine reads back](#files-the-engine-reads-back)
- [Splitting the configuration](#splitting-the-configuration)
- [Pilot: a real `x3.json`](#pilot-a-real-x3json)
- [Releases and reproducible builds](#releases-and-reproducible-builds)
- [Using x3 from another project](#using-x3-from-another-project)
- [Gaps we know about](#gaps-we-know-about)
- [Control experiments](#control-experiments)
- [The documentation gate](#the-documentation-gate)

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
| **Adoption** (`internal/adoption`) | how much of this engine the project actually runs, measured against the engine's own command table |

What `x3 scan` itself implements is a **language check**, not a behavior check.
It answers three questions about every `//x3:` line: is the type known, is the
shape right, is the scope legal. It never calls your code. Running an example
*is* implemented, but as a separate gate — [`x3 case`](#x3-case). `x3 record`
and `x3 replay` do reach behavior, but only through the HTTP surface and only
over the traffic the recording saw. `x3 guard` reaches the outside world, but
checks the environment a run is about to happen in, not your code. A green
`x3 scan` means *"your directives are well formed"*, nothing more.

## `x3 scan`

**What it catches:** a directive that no verifier knows, is malformed, or sits
in a scope where it is not legal.

```
x3 scan [-out <file>] [dir]
```

`dir` defaults to `.`. The JSON report goes to `-out` or **stdout**; findings
and the summary always go to **stderr**, so the report can be piped while the
reds stay on the terminal.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | green — every directive is well formed and in a legal scope |
| `1` | red — at least one directive failed; each is printed with `file:line` |
| `2` | usage error, or the run could not complete |

Exit codes are the same for every command. A tree with **no directives at all**
exits `0`, so an exit code alone cannot tell "everything passed" from "nothing
was checked" — read the counts too, see
[Using x3 from another project](#using-x3-from-another-project).

### What gets walked

Only `.go` files. These directory **names** are skipped: `vendor`, `testdata`,
`node_modules`, and any name starting with `.` or `_`. The skip applies to
sub-directories only — a skipped name given *as the root* is still scanned,
which is how the control samples are reached:

```
x3 scan internal/scan/testdata/green   # exit 0
x3 scan internal/scan/testdata/red     # exit 1
```

### What a run looks like

One block per red, then the summary line, on stderr:

```
bad.go:3: unknown_category: no verifier exists for kind "nope"
	found: //x3:nope:whatever
bad.go:22: unattached: the directive binds to no declaration
	found: //x3:rule:idempotent
x3 scan: 2 file(s) - 8 directive(s) - 8 red
```

Paths are relative to the scan root and always use `/`, on every operating
system.

## How a pattern is read

**What it catches:** a pattern written about lines but read against a whole
file, which silently measures almost nothing.

Most settings match a regular expression against a **file's whole text**, and
there `^` and `$` bind to a **line**:

| The pattern | Holds when |
|---|---|
| `^func main` | some line starts with `func main` |
| `\Apackage ` | the **file** starts with `package ` |
| `(?-m)^package ` | the same thing, with the mode turned off |

Line mode is the default because the cost of the other reading is not symmetric:

| The measurement wants | A collapsed pattern gives | How it shows |
|---|---|---|
| something **found** | nothing found | loud: red over an absence |
| something **absent** | nothing found | **silent: green without measuring** |
| a **number** (`count`, `cap`) | a number near zero | **silent: debt reads as repaid** |

Nobody looks at green, so the silent rows decide the default. Nothing is lost:
`\A` and `\z` always mean the ends of the file. Where the answer depends on the
reading, the finding says so rather than leaving the number unexplained:

```
"docs/list.md" counts 3, above the cap of 1; a cap takes no debt;
^ and $ read a line here, not the whole file - read the other way it would count 1
```

**Four patterns are not read this way**, because their subject is one line or
one value, not a file: `syntax` `deny`, `secrets` `patterns[].match` and
`ignore[].match`, `boxes` `markdown.moved.match`, and `arch` `literal`
`pattern`. The last is the one to read twice — a `literal` rule looks for a
**name**, so `^name$` means "the whole literal is this name".

## Scopes

**What it catches:** a directive written where it binds to nothing.

| Scope | Where you write it | Binds |
|---|---|---|
| `decl` | in the doc comment of a func, type, var, const or **import** | that one declaration; `target` names it |
| `file` | above the `package` clause | that file |
| `pkg` | above the `package` clause **in a file named `doc.go`** | the whole package |
| `unattached` | anywhere else — inside a body, or a floating comment | nothing: always red |

`pkg` is not a different syntax from `file`; the filename `doc.go` is the only
thing that separates them.

```go
// wallet.go:15 → scope "decl", target "Wallet.Add"
//x3:rule:math:commutative
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

A method's `target` is written `Receiver.Method` with pointer stars and generic
brackets stripped: `*Wallet` and `Wallet[T]` both report `Wallet`. A directive
inside a function body has nothing to attach to and is **not silently ignored** —
it is `unattached`, and red.
## The dictionary

**What it catches:** an invented directive type, or a known type written in the
wrong shape. **A type that is not in the dictionary has no verifier, and a
directive with no verifier turns the run red.**

| Directive | Valid scopes | Requires |
|---|---|---|
| `//x3:rule:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:guard:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:case: <payload>` | `decl` only | a payload that parses: `in=(...) out=...` |
| `//x3:live` | `decl`, `file`, `pkg` | nothing |
| `//x3:skip:<reason>` | `decl`, `file`, `pkg` | a reason |
| `//x3:allow:<type>:<reason>` | `decl`, `file`, `pkg` | a type **and** a reason |

After the `//x3:` prefix the rest is split on `:` into a category and its
sub-types. A **payload** is whatever follows a colon that is itself followed by
a space — `: ` — and runs to the end of the line. The payload counts toward the
required-segment count, which is what lets a reason contain spaces:
`//x3:skip:legacy-generator` and `//x3:skip: legacy generator` are both
accepted. A trailing `:` is trimmed before parsing, so `//x3:rule:` is the bare
category and red; a doubled colon (`//x3:rule::idempotent`) is red as
`empty subkind (a doubled colon)`.

### `//x3:rule:<type>[:<subtype>...]`

**Catches:** a semantic contract the code is expected to obey. Today x3 checks
only that you named a rule, in a legal scope; nothing verifies that it holds.

```go
//x3:rule:math:commutative
func (w *Wallet) Add(n int64) int64 {
```

### `//x3:guard:<type>[:<subtype>...]`

**Catches:** an invariant — a never-condition. Same shape and scopes as `rule`;
the difference is meaning, not mechanics.

```go
//x3:guard:output:non-negative
func (w *Wallet) Withdraw(n int64) (int64, error) {
```

Written above the `package` clause of `doc.go`, the same guard binds to the
whole package. There is no `guard` red sample today — its shape check is
`rule`'s code path — see [Gaps we know about](#gaps-we-know-about).

### `//x3:case: <payload>`

**Catches:** an inline example — one input and its expected output — next to the
function instead of in a test file. **`decl` scope only**: an example belongs to
one declaration.

The payload's shape is `in=(<args>) out=<want>`. The argument list may be empty,
and the closing `)` is found by **counting** rather than by taking the last one
on the line, so nested calls fit on both sides: `in=(f(1), 2) out=ErrX`.

```go
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

`x3 scan` reads the shape, not the values; [`x3 case`](#x3-case) calls the
function and compares. Both use the **same parser** — a payload one accepted and
the other read differently would be an example that goes green without running.
The same directive at package level is well formed and still red:

```
doc.go:1: scope_not_allowed: scope pkg is not allowed; valid scopes: decl
```

### `//x3:live`

**Catches:** code that talks to a real provider and costs money to exercise, so
it can be kept out of automated runs.

```go
//x3:live

package wallet
```

It takes no sub-type and no reason, so it has no shape to get wrong; its only
failure modes are `unattached` and a doubled colon.

### `//x3:skip:<reason>`

**Catches:** a deliberate exemption. The **reason is mandatory** — a silent skip
is the failure this engine exists to prevent, so a bare `//x3:skip` is red
rather than a free pass.

```go
//x3:skip: legacy generator
const legacyRate = 3
```

### `//x3:allow:<type>:<reason>`

**Catches:** a justified silence for one specific finding — the counterpart of
`skip` for scanners that flag things. It needs **two** parts: what is silenced,
and why.

```go
//x3:allow:secret:example-only
const demoToken = "not-a-real-key"
```

## Error codes

The JSON `code` field is the stable part of the output; `message` may be
reworded at any time.

| Code | Turns red when |
|---|---|
| `unknown_category` | the type is not in the dictionary — no verifier exists for it |
| `malformed` | a required sub-type or reason is missing, a doubled colon left an empty sub-type, or a `case` payload does not parse |
| `scope_not_allowed` | the type is known and well formed, but not legal in this scope |
| `unattached` | the directive binds to nothing at all |

The checks run in that order and stop at the first failure, so one directive
reports exactly one code.

## The JSON report

```json
{
  "version": 1,
  "root": "internal/scan/testdata/green",
  "files": 2,
  "directives": [
    { "file": "bad.go", "line": 6, "raw": "//x3:skip", "category": "skip",
      "scope": "decl", "target": "NoReason", "status": "error",
      "code": "malformed", "message": "skip: expected shape //x3:skip:<reason>" }
  ],
  "summary": { "ok": 7, "errors": 1 }
}
```

| Field | Notes |
|---|---|
| `version` | schema version; it goes up when a field changes meaning |
| `file`, `line` | relative to the scan root, always `/`-separated |
| `raw` | the directive line exactly as written |
| `category`, `segments`, `payload` | the parsed line; omitted when empty |
| `scope`, `target` | resolved binding; `target` only for `decl` |
| `status`, `code`, `message` | `ok` or `error`; the last two only on `error` |

Directives are sorted by file then line, and **there is no timestamp anywhere in
the report, by design**: identical sources must produce identical bytes, so a
later comparison of two runs can never raise a false red over a clock tick.

## Expectations

**What it catches:** deleting the directives as a way to go green. A tree with
no directives scans clean and exits `0` — correctly, because nothing in it is
wrong; nothing in it is checked either, and the exit code cannot tell those
apart.

```json
{
  "expect": [
    { "name": "the ledger package keeps its guards",
      "paths": ["ledger/**"], "category": "guard", "kind": "lookup", "min": 2 }
  ]
}
```

| Field | Meaning |
|---|---|
| `name` | required; the red names the expectation that was not met |
| `min` | required, at least 1 — an expectation of zero verifies nothing |
| `paths` | globs **relative to the scan root**; absent means the whole scan |
| `category` | `guard`, `rule`, `case`, ...; absent means any |
| `kind` | the first segment after the category; absent means any |

```
BLOCK expectation_not_met: the ledger package keeps its guards
	0 verified guard directive(s), the configuration requires 2
```

**Only verified directives count.** A directive the scan marked red counts as
zero — otherwise emptying a `//x3:guard` would satisfy the expectation that
exists to notice its removal. **A stale `paths` is red, not silent**: matching
nothing gives zero, and zero meets no expectation. The report is read, never
written, so identical sources still produce identical bytes.
## `x3 case`

**What it catches:** an inline example whose declaration no longer returns what
the example says — and, just as important, an example that **nothing ran**.

```
x3 case [-config <file>] [-out <file>] [dir]
```

The engine collects every `//x3:case`, builds one test per **source file**,
runs the file's tests together as their package through the Go toolchain and
compares each result. **Nothing is written into the project**: the generated
tests reach the compiler through the toolchain's *overlay*, so an interrupted
run leaves nothing behind.

### The payload

```
//x3:case: in=(<arguments>) out=<expected>
```

`in=(...)` holds Go expressions separated by top-level commas — a nested call or
a string containing a comma is one argument, not two. `out=...` holds one
expression per result, in order; a result may be skipped with `_`, but an
example whose expectations are *all* skipped is refused, because it would
compile, run, pass and prove nothing. **A method takes its receiver as the first
argument**, so value and pointer receivers both work.

Expressions compile **inside their own package**, so unexported names are in
scope. They also see **what the file they are written in imports**: an example
above a declaration in a file that imports `strings` may say `strings.ToLower(…)`
without importing anything itself. Only the imports the example actually names
are carried — an unused import is a compile error in Go, so copying the whole
list would break the package to save one example — and the name written in the
example is the name the generated test binds, so a path whose package name is
not its last path element still resolves. A name **no import of that file
provides** stays red; carrying imports is not a licence to invent them.

The expected value is never assigned to a variable first, so an untyped constant
takes the type it is measured against (`out=5` holds against `int64`). Errors
compare with `errors.Is` and then by message, so a wrapped sentinel still
matches; everything else goes through `reflect.DeepEqual`.

### Green

```go
//x3:case: in=(2, 3) out=5
func Add(a, b int) int { return a + b }

//x3:case: in=(0, 1) out=0, ErrEmpty
func Withdraw(balance, amount int) (int, error) {

//x3:case: in=(&Counter{Total: 2}, 3) out=5
func (c *Counter) Plus(n int) int {

// The file imports "time"; so does the example.
//x3:case: in=(time.Second) out=1000
func Millis(d time.Duration) int64 {
```

```
x3 case: 7 example(s) in 1 package(s) - 7 passed, 0 finding(s)
```

### Red

```
wallet.go:8 (Add): example_failed
	out[0] = 5, want 6
```

### One broken example does not blind the package

A compile error in Go is **package-wide**: the toolchain names the fault once
and nothing in that package runs. Charged as it arrives, a single mistyped
example would turn every sound example beside it red — and the table would say
"all broken" where one is. So the engine reads the line number the compiler
gives, charges the fault to the **example written on that line**, drops it, and
runs the rest:

```
broken.go:11 (Half): does_not_build
	undefined: missing
```

```
x3 case: 4 example(s) in 1 package(s) - 3 passed, 1 finding(s)
```

This matters most where examples are written in parallel: one author's error
must not hide another author's proof, because hidden work is done twice. A fault
the compiler reports **outside** every example — the package's own source does
not build — belongs to no single line and is charged to all of them, which is
the honest answer in that case.

### An example nothing ran is not a green example

The dangerous state is not the wrong answer, it is **no answer**. A package
whose test entry point returns without calling `m.Run` runs nothing, the
toolchain exits `0`, and a gate that only looked for failures would call that
green. The engine keeps the name of every example it generated and demands a
verdict for each:

```
wallet.go:8 (Add): never_ran
	nothing ran the example; the package reported no result for it
```

A skipped example is refused for the same reason — `t.Skip` is not a proof — and
a package that does not compile is named as such, so the fault is looked for
where it is.

### Findings

| Code | Means |
|---|---|
| `example_failed` | the declaration was called and the result is not what the example says |
| `never_ran` | no verdict was reported for it, or it was skipped |
| `does_not_build` | the example does not compile — charged to its own line when the compiler names one, and to every example in the package when the fault is in the package's own source |
| `malformed` | the payload has no body, or does not parse |
| `not_a_function` | the example sits above something that cannot be called |
| `in_a_test_file` | the example is in a `_test.go` file, where nothing would run it |
| `wrong_result_count` | the declaration returns a different number of values than the example expects, or a method was given no receiver |

The first three are answers the toolchain gave; the last four are refusals made
**before** anything runs.

### Settings

Optional — an example lives in the source, not the configuration:

```json
{ "case": { "exclude": ["internal/legacy/**"], "timeout": "2m" } }
```

`timeout` (default `1m`) is applied to the test binary **and** to the toolchain
call around it; only the first would leave a run that hangs downloading a
dependency waiting forever.

### The report

```json
{ "version": 1, "root": ".", "config": "x3.json",
  "findings": [ { "file": "wallet.go", "line": 8, "target": "Add",
                  "code": "example_failed", "message": "out[0] = 5, want 6" } ],
  "summary": { "files": 1, "packages": 1, "cases": 1, "passed": 0, "findings": 1 } }
```

`passed` is counted separately from `findings` on purpose: "no findings" and "no
examples" are not the same sentence.

## `x3 lang`

**What it catches:** a second language outside the comments — the author's
mother tongue leaking into identifiers, log lines and error messages.

**The dictionary runs in reverse.** There is no list of forbidden words; such a
list can only cover the language somebody thought to write down. What is known
is the **allowed** language, and every token outside it is red.

```
x3 lang [-config <file>] [-out <file>] [dir]
```

### What is checked

| Read | Not read |
|---|---|
| the package name | comments (unless `comments: "en"`) |
| every **declared** identifier — function, type, variable, constant, field, parameter, result, label, import alias | the **use** of a name declared elsewhere |
| every string constant, struct tags included | import paths |

The asymmetry is deliberate: a name is spelled once where it is declared, and a
name declared elsewhere — `fmt.Fprintf`, `pgx.Connect` — is not yours to spell.

### The token rule

Text splits on anything that is not a letter and at case boundaries, with runs
of capitals kept together: `JSONPath` → `json` + `path`, `TOTAL` → `total`. A
run of capitals stays whole on purpose — split letter by letter, a foreign word
in capitals would dissolve into fragments and slip through. Fragments under
three letters are not read. Each remaining word must be in the embedded
dictionary or in `language.allow`. On top of that, one absolute rule: **any
non-ASCII letter outside a comment is red**, and `allow` cannot excuse it.

### `language` in `x3.json`

```json
{ "language": { "allowed": "en", "comments": "any",
                "allow": ["cfg", "ctx", "dsn", "omitempty"] } }
```

| Field | Meaning |
|---|---|
| `allowed` | the language outside comments; `en` is the only embedded dictionary, and any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone; `en` holds them to the dictionary |
| `allow` | project terms no dictionary has. One ASCII word, three letters or more — an entry that could never match is rejected rather than ignored |

No `language` section is not an error; the default is `en` / `any` / no list. A
section that *is* written and is wrong stops the run — fail-closed.

### What a run looks like

```
sample.go:8:2: not_in_dictionary: notaword (identifier)
sample.go:14:18: non_ascii_letter: é (string)
x3 lang: 1 file(s) - 2 finding(s) - dictionary "en"
```

The report carries the same findings sorted by file and line, with no timestamp.
`code` is the stable part — `not_in_dictionary` or `non_ascii_letter`; `where`
is `identifier`, `string` or `comment`.

### The embedded dictionary

141,848 words are compiled into the binary (`internal/lang/english.txt`). It is
generated from the **English Speller Database** (ESDB, formerly SCOWL) at
<https://app.aspell.net/create>, size 70, US spelling, diacritics stripped, with
the `hacker` list included — which is why `http`, `auth` and `err` are already
words. Its licence requires the notice to travel with any copy:

> Copyright 2000-2026 by Kevin Atkinson
>
> Permission to use, copy, modify, distribute, and sell any part of the English
> Speller Database (ESDB, previously known as SCOWLv2), or word lists created
> from it, is hereby granted without fee, provided that the above copyright
> notice appears in all copies and that both the above copyright notice and this
> notice appear in supporting documentation. Kevin Atkinson makes no
> representations about the suitability of this database for any purpose. It is
> provided "as is" without express or implied warranty.

Do not edit the file by hand. A word that belongs to your project belongs in
`language.allow`.

## `x3 arch`

**What it catches:** the shape a project claims in prose — "the core does not
know the modules", "only the entry point wires them" — drifting from the shape
it has. The rules live in `x3.json`, the verifier in the engine: the project's
names never enter a verifier.

**All nine rule kinds are built:** `deps` with three matchers (`import` reads the
import graph, `literal` reads names the compiler never sees, `symbol` reads what
the code uses), `required`, `pairing`, `flow`, `exposure`, `duplication`,
`vocabulary`, `consistency`, `containment`. A kind with no verifier stops the run
with exit `2` — a planned kind that passed silently would be worse than no rule.

```
x3 arch [-config <file>] [-out <file>] [dir]
```

**No `arch` section means exit `2`**, deliberately the opposite of the language
gate's default: a language has a universal default, an architecture does not, and
an invented default architecture is the most dangerous silent green there is.

### `arch` in `x3.json`

```json
{ "arch": {
    "components": { "contract": ["internal/engine/**"],
                    "checkers": ["internal/arch/**", "internal/lang/**"],
                    "entry":    ["cmd/**"] },
    "rules": [
      { "name": "the-contract-knows-no-implementation", "kind": "deps",
        "match": "import", "from": "contract", "deny": ["checkers", "entry"] },
      { "name": "only-the-entry-point-reaches-a-checker", "kind": "deps",
        "match": "import", "to": "checkers", "allowFrom": ["entry"] } ] } }
```

That is this repository's own section; the engine holds itself to it on every
`check.ps1` run.

### Components

A component is a name and a set of path patterns, declared here by path rather
than labelled in the source, so the shape is reviewed in one place.

| Pattern | Matches |
|---|---|
| `**` | zero or more path elements; at the end of a pattern, **at least one** |
| `*` | a run inside one element, never crossing `/` |
| `?` | one character inside one element |

That is the whole syntax, and the omission is loud on purpose: a pattern starting
with `!` is **refused, exit `2`**, in every section that takes patterns. In most
dialects it negates; here it would be read as an ordinary name, match nothing,
and whoever wrote it would read the green as proof the exclusion worked. A blank
pattern is refused for the same reason.

A single `*` catches the component's **instance** — `internal/modules/*/**` tells
`alpha` from `beta` — which is what `except: "self"` compares. Two boundaries the
engine enforces: **paths resolve against the module root**, so checking a subtree
does not invalidate them; and **a file has one component**, because a package
silently counted on the wrong side is worse than one with no component.

### Narrowing the source set

`sources` says what a rule reads; `exclude` takes files back out and is read
**first**, so it can only narrow. Both may sit at the top of `arch`, where rules
inherit them, and on a rule, where the rule's own list **replaces** the inherited
one — otherwise a rule could never escape a pattern declared above it.

```json
{ "arch": { "sources": ["**/*.go"], "exclude": ["**/*_test.go"] } }
```

An exclusion is an escape hatch, held to the same law as an exemption: a pattern
taking **no** file out is `dead_exclusion`, one taking **every** file out is
`empty_scope`, and `"exclude": []` is exit `2` — an empty list cannot be told
from an absent one, and whoever wrote it would have silently inherited instead.
This is not `surface.exclude`, which drops paths from an `exposure` rule's
surface; `exclude` drops files from what the rule reads at all.

### The two rule forms

| Form | Written | Asks |
|---|---|---|
| outward | `from` + `deny` | may **this** component touch those? |
| inward | `to` + `allowFrom` | who may touch **this** component? |

Mixing them is refused. `except: "self"` narrows a component's ban to *other
instances of itself*; it needs a single star in that component's patterns and the
component in its own `deny` list, or it would exempt everything. In the inward
form a component's own files are always inside, and a file in no declared
component is outside.

### The `literal` matcher — names the compiler never sees

**Catches:** a table, queue or bucket name spelled by a component that does not
own it. The name is a string, the code compiles, and the failure arrives at run
time. It reads **string constants** in Go and **whole lines** elsewhere.

```json
{ "kind": "deps", "match": "literal",
  "sources": ["**/*.go", "**/migrations/*.sql"],
  "pattern": "db_(?P<owner>[a-z0-9]+)_[a-z0-9_]+",
  "owner": "modules/${owner}", "alsoAllow": ["entry"] }
```

**Ownership** (`pattern` + `owner`) derives the owner from the name itself, so no
hand-kept list goes stale; **prohibition** (`from` + `pattern`) says this
component may not spell such a name at all. Writing both is refused. Comments are
not read — what is forbidden is the *code* knowing the name — and exemptions
cover Go only, since a `.sql` file has nowhere to write one.

### The `symbol` matcher — capabilities, not layers

**Catches:** a component using a capability — encrypting, opening an outbound
request, retrying — that is one call inside a package everybody imports.

```json
{ "kind": "deps", "match": "symbol", "from": "modules",
  "deny": ["crypto/**", "net/http.NewRequest", "**/*.Retry"] }
```

**`deny` holds path patterns here, not component names.** Both the import path
and each `qualifier.Name` selector are read: an alias is resolved to its full
path, while a qualifier that is not an import is kept as written, which is what
makes `**/*.Retry` find retry logic whose package cannot be known. Catching only
the import lets a file call the function through a package it already has;
catching only the call lets the package be aliased out of sight.

### `required` — the mark every file of a class must carry

**Catches:** the seventh file somebody adds without the protection every other
file of its kind carries.

```json
{ "kind": "required", "sources": ["scripts/**/*.ps1"],
  "marker": "regex:(?m)^\\[CmdletBinding\\(\\)\\]$" }
```

The only marker form is `regex:<pattern>`, with `^` and `$` bound to a **line** —
a mark need not open the file; one that must writes `\A`. A finding carries **no
line number**: what is missing is missing from the file, not from a place in it.

### `pairing` — does anybody touch this file?

```json
{ "kind": "pairing", "sources": ["internal/**/*.go"],
  "counterpart": "sibling:*_test.go", "requires": "references-a-declaration" }
```

**Not coverage** — the question is whether any file at all names this one. In
`sibling:<template>` the `*` is the subject's own base name, so `beta.go` asks
for `beta_test.go`; read as a glob the same string says which files are
counterparts, and those are never subjects. `requires` is `exists` (default) or
`references-a-declaration`; a subject with no declarations cannot be asked the
second question.

### `flow` — where a value may appear

**Catches:** a restricted handle escaping the one place allowed to hold it.

```json
{ "kind": "flow", "sources": ["internal/**/*.go"],
  "value": { "field": "Module.pool" }, "allow": ["receiver"] }
```

The places are `receiver`, `argument`, `result`, `assignment` and `other` — the
last so an unrecognised position is refused rather than skipped. **`allow` lists
what is permitted; everything else is red**, because a deny list would leave a
place added later silently free. **It reads names, not types**: the type in
`value.field` proves only that the field is declared somewhere the rule reads,
and if it is not the rule is `empty_scope` — a renamed field must not leave a
green rule behind.

### `exposure` — what reaches the outside

**Catches:** an internal cost or margin the day the struct holding it is written
out. Import and flow rules cannot see it: the field is already in that package,
and the violation is that it leaves.

```json
{ "kind": "exposure",
  "surface": { "components": ["core", "modules"], "exclude": ["**/admin/**"] },
  "fields": ["*cost*", "*margin*"], "carrier": ["json-tag", "map-key"] }
```

`surface` minus `exclude` is how the same field stays legal on an operator screen
and illegal on a tenant one. `fields` match case-insensitively; `carrier` is a
struct field's `json:"…"` name or a string key in a composite literal. **A field
with no tag is not seen** — deciding whether it is serialized needs type
resolution, and calling every field an exposure would drown the gate.

### `duplication` — the body written twice

```json
{ "kind": "duplication", "across": "modules", "minLines": 8 }
```

Bodies are reprinted from the syntax tree with blank lines and indentation
dropped, so formatting differences disappear. Below `minLines` nothing is
compared — two modules both writing `return nil` are not a finding. The
comparison is between **instances**, so `across` needs a single star, and with
fewer than two the rule is `empty_scope`. **Identifier normalization is off**:
with it on, two deliberately separate but similar bodies would be caught too.

### `vocabulary` — the words a layer must not know

**Catches:** the core *knowing* a module without calling it — the name living in
a constant, a field name, a configuration key, a log line.

```json
{ "kind": "vocabulary", "in": "core",
  "sources": ["**/*.go", "ui/**/*.js", "config/**/*.json"],
  "terms": { "componentNames": "modules" } }
```

`terms` is either `componentNames: "<component>"` (the **instance names** are the
terms, so a new module is covered the day it appears) or `words: [...]` written
out. The tokenizer is the language gate's, so `alphaTable` is `alpha` + `table`
and a name cannot hide inside camel case. **Comments are exempt by default**;
`"comments": "checked"` covers prose too.

#### Word forms

A rename is not finished while an inflected form of the old word is in the tree,
and the list of forms cannot be kept by hand.

```json
{ "terms": { "words": ["invoice"], "match": "forms" } }
```

With `match: "word"` (default) only `invoice` is a finding; with `forms`, any
word that *starts with* the term is one — `invoices`, `invoicing`, `invoice_id` —
and the message names the term it came from. A term used this way must be at
least four characters: a short prefix falls inside innocent words.

### `containment` - a component's parts stay under its root

**Catches:** a component that is no longer movable — one part outside its root,
so deleting it leaves litter and copying it leaves the part behind.

The hard question is not where a file is but **which component it belongs to**,
and that cannot come from the directory: read that way every file is already
where it is. So ownership is declared as a **key**, a short prefix each component
puts at the start of the names of the things it owns:

```json
{ "kind": "containment", "sources": ["**"],
  "keys": { "billing": "blgx_", "orders": "ordx_" } }
```

```
apps/billing/blgx_handler.go          ok
core/blgx_helper.go                   part_outside_its_root
core/money.go                         carries no key, nobody's part
```

A key must be **5 to 10 characters** — shorter and it matches by coincidence,
longer and it is a name, which brings the coincidence back — and two keys may not
start alike. All are configuration errors. If no file carries any key the rule is
`empty_scope`: a rule that matched nothing has not passed, it did not run.

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
| `x3` | the settings file read as a **configuration** | `in-force` or `commands` |

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

### `left-exists-on-disk` — does the path still point at something?

**Catches:** a gate carrying a path constant that keeps working after the path
moves — it finds nothing, reports nothing and **exits `0`**. The most expensive
form is a criterion phrased as an absence: once the root is gone it is true
forever, and work that was never done reads as finished.

Not [`containment`](#containment---a-components-parts-stay-under-its-root): there
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
subcommands it configures; sections describing the engine's own workings (`x3`,
`update`, `baseline`, `cache`) configure no check and are in neither.
**`policy: "warn"` is not in force**, and neither is anything nested under it.

### Fields a rule has

| Field | Required | Meaning |
|---|---|---|
| `name`, `kind` | yes | unique in the file; one of the nine kinds |
| `match` | `deps` only | `import`, `literal` or `symbol` |
| `from`+`deny` / `to`+`allowFrom` | deps | the outward / inward question |
| `pattern`+`owner` / `pattern`+`from` | literal | ownership / prohibition |
| `marker` | required | the mark every file in `sources` must carry |
| `counterpart`+`requires` | pairing | the file that must name this one |
| `value`+`allow` | flow | the value to follow, and where it may appear |
| `surface`+`fields`+`carrier` | exposure | where to watch, which names, written how |
| `across`+`minLines` | duplication | the component compared with itself |
| `in`+`terms`+`comments` | vocabulary | the layer, the words, whether prose counts |
| `keys` | containment | the ownership prefix per component |
| `left`+`right`+`compare` | consistency | the two sets and how they must agree |
| `parts`+`join`/`each`, `skip`, `comments`, `syntax` | consistency | extractor details |
| `absent`, `relativeTo` | `left-exists-on-disk` | paths meant to be missing; `repo` (default) or `source` |
| `except` | no | `self` only, next to `from` + `deny` |
| `minimum` | no | the fewest subjects the rule must see |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `sources` / `exclude` | no | this rule's file set; its own list replaces the inherited one |

Configuration is validated **strictly and up front**: an unknown key, a key
belonging to another kind, a missing required key, a duplicate `name`, an
undeclared component, an unknown `policy` or an empty `rules` list stops the run
with exit `2`. An empty list is an error on purpose — a check with nothing in it
is a silent pass.

### Exemptions

`arch` adds no directive type; a violation is silenced with the dictionary's own:

```go
import (
	//x3:allow:arch: the ledger is wired to alpha here, and only here
	"example.com/app/modules/alpha"
)
```

**`skip` does not silence `arch`** — say what you are silencing by name. **An
exemption binds a line, not a tree**: above one import it covers that import,
above the `package` clause the file, and above a parenthesised `import (` block
it binds nothing and shows up dead, because a block-wide silence is a deleted
rule. A reason is required, exemptions are listed in the report separately from
violations, and a dead exemption is red.

### Scope integrity

A rule that matched nothing is `empty_scope` and red — engine behavior, not
something you choose. Every component the rule names is measured, the object side
included: a `deny` list pointing at a component with no files can never turn red.
`empty_scope` and `dead_exemption` are always `block` whatever the `policy` says;
a policy grades how bad a violation is, and neither of these is a violation —
they are the measurement failing.

**`minimum` is the floor a scan must reach.** Zero is only the last step of a
fall: a rule that read a hundred paths still reports green after a rename leaves
it three.

```
BLOCK scope_below_minimum: every-root-a-gate-names-is-still-there
  the rule saw 3 subjects and 40 were declared; a scan that shrank is a gate that stopped looking
```

`minimum` belongs to **every kind**, counting the rule's own subject. This is not
[`expect`](#expectations), which counts verified directives in a scan; this counts
what one arch rule looked at. Same disease, two organs.

### What a run looks like

```
BLOCK …/modules/beta/beta.go:3 (modules-must-not-know-each-other): forbidden_dependency
	component "modules" must not import another instance of itself
	…/modules/beta -> …/modules/alpha
x3 arch: 5 file(s) - 3 rule(s) - 1 block, 0 warn, 0 exempted
```

The same tree with `policy: "warn"` prints `WARN` and exits `0`; with the
exemption in place it prints `ALLOW …` and exits `0`.

### The arch report

No timestamp, and violations sorted by rule, then file, then line.

```json
{ "version": 1, "files": 5,
  "rules": [ { "name": "modules-must-not-know-each-other", "kind": "deps",
               "policy": "block", "subjects": 4, "violations": 1 } ],
  "violations": [ { "rule": "…", "file": "…", "line": 3, "subject": "…",
                    "object": "…", "code": "forbidden_dependency",
                    "policy": "block", "message": "…" } ],
  "exemptions": [], "summary": { "rules": 3, "violations": 1, "warned": 0, "exempted": 0 } }
```

`subject` and `object` are the packages; the message names the components.
`summary.violations` counts only `block` findings, `warned` the rest.

### Error codes

| Code | Raised by | Meaning |
|---|---|---|
| `forbidden_dependency` | `deps` | a forbidden import edge, or a name a component may not spell or use |
| `foreign_resource` | `deps:literal` | a component spelled a name another owns |
| `escaped_value` | `flow` | the value appeared where it may not |
| `exposed_field` | `exposure` | a hidden name reached the surface |
| `duplicate_body` | `duplication` | the same body in two instances |
| `foreign_term` | `vocabulary` | a layer let through a word it must not know |
| `part_outside_its_root` | `containment` | a part sits outside its component's root |
| `set_mismatch` | `consistency` | the two sets drifted; each difference is named |
| `missing_target` | `consistency` | a value read as a path leads nowhere |
| `missing_marker` | `required` | a file of the class does not carry the mark |
| `missing_counterpart` | `pairing` | no counterpart, or it names nothing from the subject |
| `empty_scope` | every rule | a component, source set or followed field matched nothing |
| `scope_below_minimum` | every rule | fewer subjects than `minimum` |
| `dead_exemption` / `dead_exclusion` / `dead_filter` | escape hatches | an exemption, exclusion or filter that took nothing out |

## `x3 freeze`

**What it catches:** a list that was only ever allowed to get shorter, growing —
the exported surface of a core package, the symbols a binary needs, the debt
somebody promised to pay down.

```
x3 freeze [-config <file>] [-out <file>] [-update] [dir]
```

```json
{ "freeze": { "baselines": [
    { "name": "core-surface",
      "sources": ["internal/core/**/*.go"], "exclude": ["**/*_test.go"],
      "set": { "from": "go", "select": "exported" },
      "file": "baselines/core-surface.json" } ] } }
```

`set` is the same extractor `consistency` uses — `go`, `json` or `regex` — so a
baseline can freeze anything a set can be read from, and it carries the same
`skip`, `comments` and `syntax` fields. `exclude` is read **first** and held to
the same law as everywhere else (`dead_exclusion`, `empty_scope`, `[]` is exit
`2`). Unlike `arch`, a baseline's exclusions are **not inherited**: baselines in
one file rarely read the same tree, and an inherited pattern would be dead — and
therefore red — for the narrow ones.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a value that is not frozen | **red** — `baseline_grew`, named |
| a frozen value that is gone | green, counted as **shrunk** |
| nothing measured at all | **red** — `empty_scope` |

`-update` rewrites the baselines and **refuses to write a set that grew**. That
refusal is the gate: an update that accepted growth would zero the baseline on
every run. Shrink is recorded, because punishing somebody for deleting dead code
teaches people to keep it.

`-update` then **measures again and reports what is left**, so its exit code
means what a plain run's does. Recording a baseline is not the same as passing:
a document above a `cap` is never written anywhere, so an update had nothing to
refuse and used to exit `0` on a tree the plain run called red. An update keeps
its report off stdout; pass `-out` for the post-update JSON.

A missing baseline file measures against an empty set — red until the first
`-update`. A baseline that cannot be measured is `empty_scope`, not a quiet pass.

### The count mode

**What it catches:** a number growing where freezing the *set* of keys would say
only "this document was already on the list", and 1471 lines turning into 1499
would pass in silence.

```json
{ "freeze": { "baselines": [
    { "name": "document-length", "sources": ["docs/**/*.md"],
      "count": { "of": "lines", "min": 1000, "max": 1500 },
      "file": "baselines/document-length.json" } ] } }
```

A baseline writes `set` or `count`, never both.

| `of` | The key | The number |
|---|---|---|
| `lines` | the file | how many lines it has |
| `matches` | the file | how many times `match` occurs, counting **every line** |
| `files` | the directory | how many files it holds |

`min` is where the gate starts looking — without it the baseline would list every
file in the repository. `max` is a **cap that takes no debt**: a key above it is
red whatever the baseline says, and `-update` leaves it out. A cap that could be
frozen would be a request, not a limit. The frozen file carries the numbers, so
a reviewer reads the debt instead of counting it.

| Measured against the baseline | Result |
|---|---|
| a key the baseline does not hold | **red** — `baseline_grew` |
| a number above the frozen one | **red** — `count_grew`, both numbers named |
| a number below the frozen one | green, **shrunk**; `-update` records it |
| a number above `max` | **red** — `above_cap`, never written |
| a key the baseline holds and nothing measures | **red** — `dead_key` |
| nothing measured at all | **red** — `empty_scope` |

The last two are where the modes part: in the set mode a value that is gone *is*
the shrink, but here a key without a number leaves a ceiling standing for a file
that may come back at its old size.

### A cap with no baseline

**What it catches:** a document that must stay short and owes nothing — a status
page, a handover note. With `count` that sentence cannot be written: putting the
cap above `min` freezes every file between the two at today's size.

```json
{ "freeze": { "baselines": [
    { "name": "status-page", "sources": ["docs/status.md"],
      "cap": { "of": "lines", "max": 300 } } ] } }
```

A baseline writes exactly one of `set`, `count` and `cap`. A key at or below
`max` is green and nothing is recorded; above it is `above_cap` with both
numbers named; nothing measured is `empty_scope`. A cap refuses `file` (nothing
is frozen), `min` (it already reports only what is above `max`) and `policy` (a
limit that can be downgraded to a warning is not a limit). **A cap is always
`block`**, and `-update` cannot reach it — but it must not therefore call the
run green, so an update reports a violated cap like any other run.

## `x3 surface`

**What it catches:** a core package's exported API changing under the
applications that import it — a parameter type widened, a return value added, a
function gone. The compiler catches it in *this* tree, on the day the whole
repository is built together; what it cannot say is **who** the change breaks,
and it says nothing at all once the callers live somewhere else. Version pinning
answers neither: it only defers the break to upgrade day, and holds the pinned
callers away from every fix in between.

```
x3 surface [-config <file>] [-out <file>] [-update] [dir]
```

```json
{ "surface": {
    "packages": ["core/**"],
    "users": ["apps/**/*.go"],
    "exclude": ["**/generated/**"],
    "file": "baselines/surface.json" } }
```

### The direction is the whole point, and it is `freeze` inverted

A debt list may only **shrink**. An API may only **grow**.

| Measured against the baseline | Result |
|---|---|
| a symbol the baseline does not hold | green, counted in `added` — a new name breaks nobody |
| a frozen symbol that is gone | **red** — `surface_removed`, with the signature it had |
| a frozen symbol whose signature differs | **red** — `surface_changed`, both signatures named |
| an `allow` entry that silences nothing | **red** — `dead_exemption` |
| a `packages` pattern that names no package | **red** — `dead_package` |
| nothing measured at all | **red** — `empty_scope` |
| **no baseline file** | **red** — `no_baseline` |

That last row is the direction's sharpest consequence and the reason it is
written down. In `freeze`, a missing baseline measures against an empty set and
everything is growth, so the run is red until somebody records it. Here growth
is *green*: against an empty set every symbol is new, the run would exit `0`,
and a gate that had measured nothing would look exactly like a gate that had
measured everything. So a missing file is red, and `-update` is what answers it.

`-update` rewrites the baseline and **refuses to write a single unexplained
break**, naming each one. Additions are recorded without ceremony.

### Deliberate breaks, and why the exemption is spent

Sometimes the API has to change. The reason is written in the configuration,
against the symbol:

```json
{ "surface": { "allow": {
    "example/core/ledger.Record": "the amount had to carry cents" } } }
```

An allowed break is a `warn`: it is counted, its reason is printed inside the
finding, and it does not stop the run. A reason is mandatory — `""` is exit `2`,
the same law every escape hatch in this engine is held to.

Then `-update` will move the frozen line for it, and the moment it does, the
`allow` entry silences nothing and the next run calls it `dead_exemption`. That
is deliberate: **an exemption is a one-time authorisation to move the line, not
a permanent hole.** The update prints every entry it spent, so the line to
delete is named rather than hunted.

### What a signature is

The identity carries no line number and no declaration order — both move
whenever anybody edits the file, and a moving identity reports a contract as
broken that nobody touched. A symbol is `<import path>.<name>`, a member is
`<import path>.<Type>.<name>`, and the value is what a caller can see:

| Written | Frozen as |
|---|---|
| `func Record(id string, amount int) (Entry, error)` | `func(string, int) (Entry, error)` |
| `func (e *Entry) Add(n int) error` | `method(*Entry) func(int) error` |
| `type Entry struct{ Total int }` | `type struct` **and** `Entry.Total` → `field int` |
| `type Reader interface{ Read() error }` | `type interface` **and** `Reader.Read` → `method func() error` |
| `type ID string` / `type Alias = other.T` | `type string` / `alias other.T` |
| `const Max = 100` / `var Default *Entry` | `const` / `var *Entry` |
| `func Map[T any](in []T) []T` | `func[T1 any]([]T1) []T1` |

Three things are normalised away, because changing them changes nothing a caller
sees, and a gate that reddens for them is switched off in its first week:
**parameter names** (the type and the position are the contract), **import
aliases** (a qualifier is written as the full import path, so `clock.Duration`
and `time.Duration` are one signature), and **type parameter names** (rewritten
to their position). Two declarations of one name under mutually exclusive build
constraints are frozen as both, joined by ` | `; picking one would blind the
gate to the other.

Two exclusions are the **language's** rule and not a setting: a `_test.go` file
is never part of a package's importable surface, and a path with an `internal`
element is already closed to the outside. Letting either in would make removing
a symbol nobody can call count as a break. A `packages` pattern that names only
internal packages is therefore `dead_package` rather than a silent nothing.

### Who breaks

The question behind the gate is not *"what changed"* but *"who breaks"*, so
`users` declares where the callers live and every finding carries them:

```
BLOCK surface_changed: example/core/web.WriteError
	example/core/web.WriteError changed from "func(net/http.ResponseWriter, int, string)"
	to "func(net/http.ResponseWriter, int, string, ...string)"
	used by (symbol): apps/one/handler.go, apps/two/panel.go
```

`usersBasis` in the JSON says what the list is, and it is not always the same
question:

| `usersBasis` | What the list holds |
|---|---|
| `symbol` | files whose code writes this exact qualified name — **the callers** |
| `type` | files that name the **owning type** of a changed member |
| `unmeasured` | `users` was not declared; nothing was read |

The `type` basis is a floor, not the set. Resolving `v.Method(…)` to the type of
`v` needs a type checker, and this engine reads syntax; so for a field or a
method the honest answer is *"these files name the type"*. It can miss a caller
that receives the value without ever writing the type's name, and it never sees
a dot-import. **A lie about who breaks would be worse than the gap**, so the
basis is written next to the list rather than left to be assumed.

## The finding baseline

**What it catches:** the thousand findings a new gate produces on its first run,
which get it switched off by the afternoon. The way out is `freeze`'s, one level
up: write down what the tree owes today, and demand that no new debt appear.

```json
{ "baseline": { "dir": "baselines" } }
```

The configuration declares a **directory**; the file name comes from the
command, so `x3 comments` reads `baselines/comments.json` and `x3 secrets` reads
`baselines/secrets.json`. Declare nothing and there is no baseline: every
finding is red.

| Flag | What it does |
|---|---|
| `-baseline <file>` | read this file instead of the derived one |
| `-update-baseline` | rewrite to this run's findings; growth is never written |

### The identity carries no line number

A baseline keyed by line number moves the day somebody adds an import: the same
finding, the file shifted under it, and a violation nobody introduced. Two of
those and the baseline is refreshed out of irritation. So a finding is
identified by **what it is, where it is, and what it says**:

| Command | A finding is identified by |
|---|---|
| `comments` | the rule and the file |
| `secrets` | the rule, the file, the pattern and the **masked** sample |
| `arch` | the code, the file, and the rule, subject and object |
| `lang` | the code, the file, and the token with the place it sits in |
| `syntax` | the code, the file and the name of the check |

```json
{ "version": 1, "command": "comments", "count": 2,
  "findings": [ { "id": "4ace75caf575", "rule": "block_too_long", "path": "a.go" } ] }
```

The **digest**, not the text, is what the run compares — a baseline storing the
matched text would put the very value `secrets` masks into a file the repository
keeps.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a finding the baseline holds | green, counted in `baselined` |
| a finding the baseline does not hold | **red** — this is the gate |
| a baseline entry the run no longer produces | **red** — `dead_baseline` |

`-update-baseline` **refuses to write a set that grew**, naming every finding
that blocked it; without that refusal the flag would turn any red run green. A
missing baseline file measures against an empty set; a file that exists and
holds nothing is a project declaring it owes nothing, and can only shrink. **An
empty file is a statement, a missing file is a beginning.**

### What can never enter a baseline

- **Warnings** — an observation is not a debt, and freezing one turns into a
  `dead_baseline` red the moment somebody fixes it.
- **Scope-integrity findings** — `empty_scope` says the rule measured nothing;
  freezing it paints a gate that checks nothing green.
- **Dead markers** — `dead_exemption`, `dead_exclusion`, an uninstalled parser.
  They belong to the gate's own health, not to the source.

## `x3 docs`

**What it catches:** a change that travelled alone — code without its
documentation, a migration without its release note.

```
x3 docs [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

```json
{ "docs": { "rules": [
    { "name": "code-changes-carry-documentation",
      "when": ["internal/**", "cmd/**"], "then": ["docs/**"] } ] } }
```

That is this repository's own section, and its own gate runs this command
against itself on every `check.ps1`.

### What counts as changed

| `-scope` | Reads |
|---|---|
| `auto` (default) | the working tree when it is dirty, the last commit when it is clean |
| `working` | `git status`, untracked files included; a rename counts as its new name |
| `head` | the files in `HEAD`, with the commit body as the place a reason may live |

The default is not a convenience: checking the last commit while the tree is
dirty would count documentation that has not been written yet. **A directory
that is not a repository is red**, not green.

### Exemption, with a reason

```
docs: none - wording of one stderr line; the capabilities document does not quote it
```

The marker is `docs: none` unless the rule says otherwise; in `head` scope it
lives in the commit body, in `working` scope it is passed with `-reason`. **The
marker alone is red** — `exemption_without_reason` is a separate code from the
missing change, because an exemption nobody had to justify becomes the only path
within a month.

## `x3 secrets`

**What it catches:** a credential in the source — the one mistake that cannot be
taken back, because it is in the history and the history is shared.

```
x3 secrets [-config <file>] [-out <file>] [dir]
```

**No `secrets` section is not an error** — the builtin patterns apply. A leak
scan is not a check somebody skips by not configuring it.

```json
{ "secrets": { "sources": ["**"], "exclude": ["internal/secrets/testdata/**"],
    "patterns": [ { "name": "internal-service-token", "match": "svc_[0-9a-f]{32}" } ] } }
```

`builtin: false` turns the shipped patterns off, and then the project must write
its own — a scan with no patterns is refused rather than passed.

### The report carries no secret

**The value found is masked**: the first four characters, then its length.

```
BLOCK config.go:7: secret_found
	a value matching "aws-access-key" is in the source
	value: AKIA... (20 characters)
```

A report is written to a log, pasted into a ticket, shared on a screen. A tool
that found a leak and then printed it would be a second leak.

### The shipped patterns

`private-key-block` (a PEM header of any kind), `aws-access-key`
(`AKIA`/`ASIA`), `google-api-key` (`AIza`), `slack-token` (`xox[baprs]-`),
`github-token` (`gh[pousr]_`), `json-web-token` (a three-part JWT), and
`url-with-password` (a password inside a connection string).

**Every pattern is a recognisable format, not an entropy score.** A value that
looks random is not thereby a secret, and a gate that reds every random-looking
string is switched off within a week. Binary files are skipped for the same
reason.

### Excluding what the pattern also catches

**What it catches:** the noise that gets a real gate switched off. A pattern for
an IPv4 address also matches a private range, an RFC 5737 documentation address,
a browser version like `126.0.0.0`, and a date written with dots. Measured on a
real production Go application, the bare patterns for its own credential shapes
returned **530 findings where 63 were real**.

```json
{ "name": "ipv4-address",
  "match": "(?:^|[^0-9.])((?:[0-9]{1,3}[.]){3}[0-9]{1,3})(?:[^0-9.]|$)",
  "ignore": [
    { "match": "^(?:0|10|127)[.]", "reason": "this host and the private range" },
    { "value": "255.255.255.255", "reason": "the broadcast address" } ] }
```

An entry writes **`match`** or **`value`**, never both, and always a **`reason`**.
It reads the **value that was found**, not the line — excluding by line would
hide every other value sharing it; `"on": "match"` hands it the whole match
instead, for when the surroundings decide rather than the value. An exclusion
that excluded nothing is `dead_ignore` and **red**.

**A capture group is the value.** A pattern usually matches the characters
around a value — a separator, a boundary — and those are not part of the secret,
so the mask covers the group and the exclusions read the group. Every match on a
line is examined, not only the first.

**Lookaround does not exist here.** Go's engine is RE2, so `(?<!...)` comes back
as `invalid named capture`, which sends the reader hunting for a group nobody
wrote. The engine names the real gap and points at what replaces it:

```
patterns[0]: match: error parsing regexp: invalid named capture: `(?<![0-9.])[0-9]{15,17}`;
"(?<!" is a lookaround and RE2 has none - write the exclusion as an ignore entry instead
```

Exclusions belong to the pattern that carries them, so the shipped patterns
cannot take one.

### Exemption, with a reason

```go
//x3:allow:secret: a documented example key, not a live credential
const example = "AKIAJ4EXAMPLEKEY9ABC"
```

The directive covers **its own line and the one below it**, and must be the
first thing on its line after the comment opener — `#`, `--`, `/*`, `*`,
`<!--`, `;` — so a sentence that merely *mentions* it is not one. That is not
theoretical: this package's own comments describe the directive, and the first
scanner read them as exemptions. An exemption nobody needed is `dead_exemption`
and red; the example above carries a real-shaped key on purpose, because written
with an ellipsis the exemption over it would cover nothing and this document
would fail the scan it describes.

## `x3 comments`

**What it catches:** a comment block that grew past reading length — not
documentation, but a document in the wrong place.

```
x3 comments [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [dir]
```

**Block length is red.** Consecutive comment lines form a block; a blank line or
a line of code closes it.

```
BLOCK internal/source/glob.go:10: block_too_long
        a comment block runs 11 lines, the limit is 10; what needs this many
        lines belongs in a document
```

**The ratio only warns.** When a file carries more comment lines than code
lines, the run says so and stays green. The measure is *necessity*, not count,
and a gate that failed on a ratio would make people delete comments that were
needed. It speaks only above `ratioFloor`, because a three-line file with four
comment lines is not a finding.

```json
{ "comments": { "block": 10, "doc": 20, "ratio": "warn", "ratioFloor": 30,
                "docByExtension": { ".go": 20 }, "openers": { ".ts": "//" } } }
```

The **opening block** — everything before the first line of code — has its own
limit, twice the ordinary one by default: it is read once and describes the
whole file. "The first block" would have been the wrong rule, because the first
block *inside* the code is an ordinary block. Not every language earns the same
allowance, so `docByExtension` writes it per language.

Lines that talk to a tool rather than a reader are neither prose nor code —
`//go:...`, `// Deprecated:`, `//nolint`, `#!`, `// +build`, and x3's own
directives. They close a block and count for nothing.

Ten languages are built in (`.go`, `.js`, `.java`, `.cpp`, `.py`, `.ps1`,
`.yaml`, `.yml`, `.sql`, `.lua`); an extension that is not among them is skipped
rather than guessed at, and a project adds its own with `openers`.

An exemption carries a reason and a dead one is red. It may sit **above or
below** the block it covers: above is natural, but `gofmt` moves directives to
the end of a Go doc comment, and a rule accepting only one side would break
itself on the next format.

```go
//x3:allow:comments: the glob syntax table is the contract itself
```
## `x3 boxes`

**What it catches:** a work list drifting — work that finished while the box
stayed open, and a box closed by somebody who only meant to finish it. Each box
carries the criteria that would prove it done, and the gate asks both questions.

```
x3 boxes [-config <file>] [-out <file>] [dir]
```

```json
{ "boxes": [
    { "id": "arch-containment", "title": "the containment rule kind", "state": "open",
      "done": [ { "when": "file", "path": "internal/arch/containment.go" },
                { "when": "pattern", "sources": ["docs/CAPABILITIES.md"],
                  "match": "### `containment`" } ] } ] }
```

The list lives in its own file (`{ "boxes": { "file": "docs/OPEN-WORK.json" } }`)
because it changes weekly while the configuration changes yearly. That is this
repository's own list, and `check.ps1` runs this command against it.

### A box that is not needed yet

Some work is owed only once something else happens — a second tenant, a version
bump that has not landed. `when` says what makes it due:

```json
{ "id": "second-voice-application", "state": "open",
  "when": [{ "when": "pattern", "sources": ["apps/*/kind.json"], "match": "\"voice\"" }],
  "done": [{ "when": "file", "path": "docs/migration-b.md" }] }
```

While the condition does not hold the box waits: open without being red, and
counted. The rule that does **not** relax is the other one — **a box closes
because the work was done, never because it stopped being needed.**

### Both directions, or neither

| State | Criteria | Result |
|---|---|---|
| `open` | all met | **red** — `box_finished`: the work is done, the list is stale |
| `done` | any unmet | **red** — `box_unproven`, each unmet criterion named |
| `open` | some unmet | green |
| `done` | all met | green |

Asking only the second lets a list fill with finished work; asking only the first
leaves closing without proof free.

### The criteria

| `when` | Fields | Holds when |
|---|---|---|
| `file` | `path` | `path` exists |
| `pattern` | `sources`, `match` | `match` is found under `sources` |
| `absent` | `sources`, `match` | `match` is found **nowhere** under `sources` |
| `sql` | `dsnEnv`, `query`, `equals`, `driver`, `timeoutMs` | the query's first cell equals `equals` |
| `command` | `command`, `args`, `output`, `timeoutMs` | it exits `0` **and** its output meets `output` |
| `manual` | `by`, `seen`, `signed` | `signed` is written |

`match` is read with `^` and `$` bound to a **line**
([how](#how-a-pattern-is-read)). The sharp edge is `absent`: a `pattern` that
stops matching goes loudly red, but an `absent` that stops matching goes **green
without measuring anything**. It is also what makes deletion provable — "the old
call site is gone" is exactly the sentence that becomes true when that work
finishes.

`equals` is required on `sql`, because a query that only has to *run* is answered
by an empty table; the DSN is read from the environment by name and never written
in the list, and a `driver` this binary never registered is a configuration error
(exit `2`) rather than a criterion that quietly could not be measured. On
`manual`, `by` and `seen` are required because a criterion without them is an
intention, and every run prints how many criteria are manual — a ratio that grows
is a list drifting back to nobody checking. Fields belong to exactly one
criterion, and a foreign one is refused rather than ignored in silence.

#### The exit code is half a command criterion

A runner asked for a test that was never written exits `0`; a suite whose every
case skipped itself prints `PASS` and exits `0`. Both read as "finished", which
means **writing the criterion is enough to close the box**.

```json
{ "when": "command", "command": "go", "args": ["test", "-v", "./..."],
  "output": { "must": ["--- PASS"], "mustNot": ["no tests to run"],
              "retry": { "when": ["no tests to run"], "args": ["-tags", "integration"] } } }
```

`must` and `mustNot` are plain substrings searched in **both streams together** —
a runner's "I found nothing to run" is usually on stderr. `retry` runs the
criterion **once** more, only when the first output says one of `when`, and the
second run *replaces* the first: "either attempt may hold" would be an escape
hatch. Nothing here is specific to any runner.

#### A criterion that cannot run

A `sql` criterion whose DSN is empty, or a `command` that cannot start, is
**unmet** — never met — with the reason beside it, and counted in
`summary.unmeasured` and on the human line, because a count that is always above
zero is a gate that never actually runs. A list with no boxes is `empty_scope`: a
list that says nothing does not say everything is finished.

### A list written as a document

Most projects keep their open work in their documents, as Markdown checkboxes
with more than two states. `sources` reads the list that way — and `file` and
`sources` cannot both be written.

```json
{ "boxes": { "sources": ["docs/**/*.md"], "markdown": {
    "states": [
      { "mark": " ", "name": "open",  "means": "open" },
      { "mark": "~", "name": "doing", "means": "open", "requires": ["done-so-far", "left"] },
      { "mark": "x", "name": "done",  "means": "done" } ],
    "criterion": { "key": "criterion", "kinds": {
      "exists":   { "when": "file" },
      "contains": { "when": "pattern" },
      "passes":   { "when": "command", "prefix": ["go", "test", "-v"],
                    "output": { "must": ["--- PASS"] } },
      "by-hand":  { "when": "manual", "separator": " - ", "signed": " - signed " } } },
    "minLength": 4 } } }
```

```markdown
- [~] the report that names a state
      done-so-far: the reader is written
      left: the report still prints only the mark
      criterion: exists docs/reader.md
```

#### The engine does not know what "in progress" means

A mark gets a `name`, which the report uses, and a `means`, which is the only
thing the engine acts on: `open` (all criteria met is red), `done` (any unmet is
red), `silent` (neither direction is asked). So whether "waiting on somebody"
goes red when its proof already stands is one line of the project's
configuration. `silent` is an escape hatch, and a `silent` state no box carries
is `dead_state` and red.

#### The record a state must carry

An in-between state is a claim, not a condition. `requires` names the fields that
must sit in the item's body, in the project's own words; a missing one is
`box_record`, and so is a field so short it is a way of not answering —
`minLength` sets the floor, and `left: -` does not clear it. A field line may
carry any leading decoration; what counts is a name, a colon and something after
it. The body of an item is everything indented under it, or — for an item written
as a heading — everything to the next heading. Checkboxes inside a fenced code
block are examples, not work.

#### Writing a criterion in prose

`criterion.key` opens a criterion line and `kinds` maps the project's word onto
one of the six criteria. Everything a criterion needs but a document should not
repeat — the DSN variable, the runner `prefix`, the separators — lives in the
kind, not in the line.

| `when` | The rest of the line is read as |
|---|---|
| `file` | a path |
| `pattern`, `absent` | a place, then the expression; the place matches the file **and** everything under it |
| `sql` | the query, `==`, the value it must give |
| `command` | arguments appended to `prefix` |
| `manual` | who looks, the separator, what they must see |

Place and expression split at the first space, so a path containing one is
quoted, and **a quote that never closes is a configuration error** on that line
rather than a path quietly split in two:

```
criterion: contains "docs/design notes/READER.md" the parser is here
```

#### What the reader refuses

`box_unknown_state` is a mark the configuration never declared. `box_unlisted` is
`* [ ]`, `+ [ ]` or `1. [ ]` — drawn like a checkbox, collected by nothing, which
is the quiet one: the work was written down and is in no list, so nobody will
come looking for it.

#### Regions that are not work

A note that keeps a work list usually also shows **how an item is written**, drawn
with the same checkboxes. Fenced code blocks are skipped because that is
Markdown's own writing; every other marker is declared:

```json
"markdown": { "examples": [ { "open": "^<!-- EXAMPLE -->", "close": "^<!-- /EXAMPLE -->" } ] }
```

Both are required, and the marker lines are skipped with everything between them.
A region that opens and never closes is `example_unclosed`; one that opens in no
document at all is `dead_example`.

### A list that is adopted gradually

A gate that reds a thousand times on its first run is switched off on its second,
so `boxes` reads a baseline exactly like `comments` or `secrets`:

```
x3 boxes -baseline baselines/boxes.json -update-baseline   # freeze what stands now
```

What may be frozen is **how the list is written today** — `box_uncovered`,
`box_record`, `box_unlisted`, `box_unknown_state`, `box_owner`, `box_moved`,
`box_suspect`. What may **never** be frozen is what the list *claims*:
`box_finished` and `box_unproven` (freezing them makes finished work sit open
forever and closing without proof free), `empty_scope`, and the gate's own health
codes. A baseline buys time to write the criteria; it does not buy permission to
stop asking the two questions.

### A list that is finished, and where it goes next

Every criterion so far reads one **item**. This one reads a whole **document**,
because a list can fail as a list while every item in it is written correctly.

```json
{ "markdown": { "retire": { "from": "ongoing", "to": "done",
    "bare": true, "empty": true,
    "few": [ { "open": 2, "finished": 1 }, { "open": 5, "percent": 75 } ] } } }
```

`from` is the part of a path that says "this document is open work", `to` is what
replaces it, and **both are the project's words**. `bare` finds a document among
the open lists carrying no box at all (`list_has_no_box`), `empty` one where
nothing is open any more (`list_finished`), and `few` one where a threshold holds
(`list_nearly_finished`).

`bare` is why the rule walks **documents** rather than boxes: a file with no box
produces nothing to walk past, so the one failure that leaves work completely
invisible is exactly the one a box-by-box gate cannot see — and its finding does
not say "move this", because what it needs is boxes. Each `few` threshold is "at
most this many open, and this much finished", and **the finished half is
required**: `open` alone would retire a one-item plan nobody has started, filing
work nobody has begun under work that is done.

**The engine does not write the list out.** Turning the report into a page
somebody reads over breakfast is formatting, and a gate that also published
documents would own two contracts.

### Work that moved rather than finished

An item is sometimes closed because it was written down somewhere else: the work
did not finish, its **place** changed.

```json
{ "markdown": { "moved": { "match": "^\\s*moved to `([^`]+)`\\s*$", "roots": ["docs"] } } }
```

An item whose body carries that line is outside **both** directions — its criteria
are not run, and it is not `box_uncovered` either. The pattern is the project's
own and its **first capture group** is where the work went. It is not free: saying
"moved" would otherwise be the cheapest way to silence a criterion, so the target
has to **exist**, and one nothing can be found at is `box_moved`. A marker that
matches nothing is **not** red — the item simply keeps its criteria, so the
failure is loud rather than blind.

### A scope that can be narrowed

```json
{ "boxes": { "sources": ["docs/**/*.md"], "exclude": ["docs/external/**"] } }
```

The law is `arch`'s and `freeze`'s, shared in one place: a pattern that takes no
document out is `dead_exclusion`, an empty list is refused, and a scope holding no
box is `empty_scope`. `exclude` narrows documents, so it may not be written beside
`file` — a machine-written list has no scope to narrow.

### A document that can carry a signature

A `manual` criterion holds when `signed` records that the looking happened.

```markdown
- [x] the installer works on a clean machine
      criterion: by-hand the release owner - the installer runs on a clean machine - signed 2026-09-07
```

`separator` splits who looks from what they see; `signed` splits what they see
from the record that they did. Both are the project's own words, and if the kind
declares no `signed` marker a manual criterion can never hold — correct rather
than convenient.

### A criterion that stopped measuring

The quietest way a work list dies is criteria that cannot fail. Three writings do
it, all three go green, and none measures anything:

```json
{ "boxes": { "suspect": { "repeat": 3, "always": ["go.mod", "README.md"],
                          "selfProof": true } } }
```

`repeat` finds one criterion carried by that many items or more; `always` a
criterion pointing at a path the project carries in **every** state; `selfProof` a
criterion whose scope is the very document the item is written in. Each is
`box_suspect`. `repeat` counts distinct items and skips `manual` criteria, and is
refused below `2`; `selfProof` asks the scope matcher, so a glob that reaches the
document is as visible as a path that names it. The section must ask for at least
one of the three.

### Who a manual criterion may wait on

```json
{ "boxes": { "manual": { "denyBy": ["the person who owns this project"] } } }
```

A `manual` criterion whose `by` matches one of those names is `box_owner`. This is
a **prohibition**, not an escape hatch, so it does not shout when it matches
nothing — a rule that catches nothing is good news.

## `x3 syntax`

**What it catches:** a file no compiler reads going out broken — a template, a
settings file, a script the browser loads at run time. The server still answers
200 and the screen is simply blank.

```
x3 syntax [-config <file>] [-out <file>] [dir]
```

The gate does not guess which parser a file wants; a project declares it, and
each check is exactly one of three kinds:

```json
{ "syntax": { "checks": [
    { "name": "every-settings-file-parses", "sources": ["**/*.json"], "as": "json" },
    { "name": "browser-scripts-parse", "sources": ["ui/**/*.js"], "run": ["node", "--check"] },
    { "name": "no-escaped-quote-in-an-attribute", "sources": ["ui/**/*.html"],
      "deny": "=\"[^\"]*\\\\'",
      "reason": "a backslash escape inside an attribute is not valid here" } ] } }
```

`as` names a parser the engine carries — `json` is the only one, because a
format half-understood is worse than one not understood at all. `run` names an
external parser: the path is appended, and a non-zero exit is a finding carrying
the parser's own first line. `deny` is the other half of the same problem — text
that parses but means nothing in this format — and it needs a `reason`.

**A parser that is not installed is red.** A gate that quietly skips its check
on a machine without the tool reports green having verified nothing. A project
that genuinely wants it optional writes `"missing": "warn"`. A check whose
sources match nothing is `empty_scope`.

## `x3 scope`

The other half of the coupled-change problem. `x3 docs` asks what a change must
bring **with** it; this one asks what it must **stay away from**. A part that
ships on its own stops shipping on its own the day it rides in the same commit
as something else.

```
x3 scope [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

A lane is a set of paths, plus the paths allowed to travel with them:

```json
{ "scope": { "lanes": [
    { "name": "site", "paths": ["web/site/**"], "also": ["docs/site/**"],
      "exempt": "lane: crossed" } ] } }
```

A change that touches nothing in `paths` is none of this lane's business; one
that does must stay inside `paths` and `also`, and anything else is named file
by file:

```
BLOCK site: outside_the_lane
        this change is in the "site" lane (1 file(s)) and also touches 1 file(s)
        outside it; split the change, or say "lane: crossed" followed by a reason
        outside: internal/core/money.go
```

Crossing is allowed when it is said out loud — in the commit message, or
`-reason` for a working-tree run — and a marker with nothing after it is red.
`-scope auto` reads the working tree when it is dirty and `HEAD` when it is
clean. Both this gate and `x3 docs` read the diff through the same code, because
two gates disagreeing about what a commit touched would each be right about a
different commit.

### A lane a branch declares

The lane above is opened by the **files**. That is the wrong reading for a
working lane — a branch that exists so two people can work without landing on
each other. There the promise is **"this branch only ever works here"**, and a
commit that touches nothing inside the lane and everything outside it is the
violation itself.

```json
{ "name": "site", "branch": "work/site*", "base": "main",
  "paths": ["web/site/**"], "exempt": false }
```

Three things change: **the branch opens the lane, not the files**, so on any
other branch the lane is silent; **the whole branch is measured**,
`merge-base(base, HEAD)..HEAD`, so a violation in the first commit does not go
out of sight once a clean commit is put on top of it; and **reading the range is
part of the answer** — if git cannot be read, or the base is not there, the run
is red.

`exempt: false` closes the lane: the marker is not even looked for. That is a
deliberate hole in the rule everywhere else in the engine, that an exemption
must be sayable. Write it only when the boundary is somebody's stated condition
rather than a convention. `base` without `branch` is refused.

## `x3 test`

**What it catches:** a suite that grows with the repository until nobody runs
it. `x3 test` hands the runner only the units a change can reach, so the cost of
a run follows the **change**, not the size of the tree.

```
x3 test [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [-scope auto|working|head] [-reason <text>] [-full] [dir]
```

The engine knows no test runner: the command, the way a unit is written on the
command line, and the way a dependency is read all come from the configuration.

```json
{ "test": {
    "units": ["cmd/*/*.go", "internal/*/*.go"],
    "tests": ["**/*_test.go"],
    "ignore": { "docs/**": "the published documentation is compiled into nothing a test exercises" },
    "module": "x3",
    "imports": { "from": "regex", "select": "^\\s*(?:[\\w.]+\\s+)?\"(x3/[^\"]+)\"",
                 "comments": "exempt" },
    "run": { "command": ["go", "test"], "package": ["./{unit}"], "all": ["./..."] } } }
```

`units` declares which files **define** a unit; the unit is the directory
holding them. `package` is that unit's argument list — a list, because one runner
takes `./pkg` in one word and another takes `-p pkg` in two — and `{unit}` must
appear in it, or every unit would render the same argument and the selection
would be a lie. `imports` is the ordinary extractor with one capture group;
`module` is the prefix meaning *this repository*, and anything outside it never
enters the graph. `comments: "exempt"` earns its line: an import inside a
commented-out block never runs, and an edge drawn from it would drag a unit into
every run for nothing.

### A test's import does not travel

A test file's dependency belongs to **that unit alone**: a test binary links it,
the package does not, so an importer of that package never sees it.

This is not a detail. In the pilot, `database`'s test imports a shared helper,
that helper imports the engine package, and `ledger` imports `database`. With
test edges travelling, touching one leaf reached **54 of 85 packages**; with
`tests` declared, **17** — and the 37 others had no path to the change at all.

A `tests` pattern that matches nothing is *not* red, and the asymmetry is
deliberate: a dead `ignore` makes a run **narrower** than the tree justifies, a
dead `tests` pattern only makes it **wider**. The dead-escape-hatch law guards
the direction that can hide a failure.

### Which unit a file belongs to

| The file | Its unit |
|---|---|
| matches `units` | its own directory |
| does not match `units` | the nearest unit **above** it |
| has no unit above it | none: the run goes **full** |

The search upward never reaches the repository root. If it did, the root would
be every file's ancestor, and a change to a README would narrow the whole tree
down to one package — the quietest possible narrowing, exactly where we set out
to prevent it.

### Fail-closed: when the run goes full, it says so

| Reason | What happened |
|---|---|
| `orphan` | a changed file is in no unit and no `ignore` covers it |
| `graph` | a dependency resolves inside `module` but to no unit |
| `diff` | the change could not be read from git |
| `units` | the `units` patterns match no file |
| `forced` | `-full` was written |

A full run is not a fault and does not turn the command red — the engine says it
could not narrow, and does the work anyway. What *is* red is a dead `ignore`
pattern or a dead `skip` inside `imports`. `ignore` is measured against the whole
tracked tree, not this run's changed files: not having been touched today does
not make a pattern dead. Every entry carries a reason, because "this path cannot
change behavior" is a claim and a claim wants an owner.

### The cache

Each unit's result is stored under a digest of **everything its test binary
links** — its own files, everything it imports transitively, and what its test
files import. Selection and digest read the same set, so a unit that was not run
can never be sitting on a stale green.

**Only green is stored.** Which unit failed inside a batched run can only be read
out of the runner's output, and that output is specific to one language; a red
result simply runs again, which is what happens anyway while it is being fixed.
A **full** run does not consult the cache at all: the reason it went full is that
the effect of the change could not be computed, and what cannot be computed
cannot be looked up. The engine never adds `-count=1`, which would defeat the
runner's own cache.

### A monorepo with more than one module

```json
"run": { "dir": "backend", "command": ["go", "test"], "package": ["./{unit}"], "all": ["./..."] }
```

`run.dir` launches the runner somewhere other than the run root. Units still
carry their repository-relative names (`backend/internal/auth`), so the changed
set and the graph stay in one coordinate system and only the command line is
rewritten. Every module gets its own `test` section.

### The measurement — and what it honestly shows

Pilot: a production Go repository, one module of 85 packages, read-only. One
leaf package touched; 17 of 85 affected.

| Case | the runner alone | `x3 test` | ratio |
|---|---|---|---|
| cold build cache (a fresh CI machine) | 28.7 s | 24.0 s | 1.2× |
| warm cache, one package touched | 12.3 s | 12.1 s | 1.0× |
| nothing changed since the last run | 4.15 s | **0.34 s** | **12×** |
| everything, `-count=1` | 17.8 s | same under `-full` | 1.0× |

**Read the table honestly.** For Go, `go test ./...` is *already* incremental at
the same granularity this command selects at, so narrowing the package list buys
almost nothing on top of it. The one place it wins outright is the run where
nothing changed: the runner still walks all 85 packages to decide it has nothing
to do, and that walk grows with the repository. For a runner **without** a cache
of its own — most of them — the first three rows would look very different. The
engine does not assume either case; it measures.

## `x3 record`

**What it catches:** nothing on its own — it produces the source a later run is
compared against. Every other checker here reads the project **at rest**; this
one reads it **in motion**, standing in front of the running application as a
reverse proxy, passing traffic through untouched and writing down what went by.

```
x3 record [-config <file>] -listen <addr> -target <url> -ledger <file>
```

The application is not modified, not rebuilt and not linked against x3 — no
middleware, no import, no build tag — which makes the capability
language-independent from the first line. Exit codes: `0` on a clean shutdown,
`2` for usage, configuration or I/O errors. **There is no `1`**: recording is not
a gate.

### The ledger

One file per suite, JSON Lines, one interaction per line:

```json
{"n":1,"req":{"method":"POST","path":"/orders","headers":{"Content-Type":["application/json"]},"body":{"name":"a cup"}},"res":{"status":201,"body":{"id":"17","state":"created"}},"ms":34}
```

One line per interaction is deliberate: a behavior change then shows up as a
**diff a human can read** in review. `n` is the recorded order, and replay
follows it. Header and query values are kept as lists, because a header folded
into one string comes back different when it is sent again. A JSON body is
stored parsed, so a change inside it reads as one changed field; anything else is
text. `ms` is written for the reader — nothing compares it.

The ledger is a source file: committed, reviewed, and the thing that shrinks a
pile of hand-written behavior tests. Which is exactly why nothing secret may
reach it.

### Redaction happens before the disk

A secret that was never written cannot leak later, so redaction sits between
reading the response and writing the line. Three layers, the first two needing no
configuration:

1. **Credential headers**, always: `Authorization`, `Cookie`, `Set-Cookie`,
   `Proxy-Authorization`. What they carry is identity, not behavior.
2. **The `secrets` pattern set**, applied to every recorded value.
3. **The project's own field paths**, under `record.redact`.

```json
{ "record": { "redact": [
    { "path": "res.body.token", "reason": "session token" },
    { "path": "res.body.items.*.email", "reason": "personal data" } ] } }
```

A rule without a reason is refused. A path starts with `req` or `res`, then
`headers`, `query` or `body`; `*` means every element of an array or field of an
object. A path that reaches nothing is not an error — that field did not appear
in this run — but a path that **cannot mean anything** (`res.query.page`) is a
configuration error, because a misspelled rule would otherwise look exactly like
a rule with nothing to hide.

Hidden values are written as `"<redacted:reason>"`, so a reader can tell a masked
field from an absent one. **The proxy stays transparent**: the client receives
the application's answer exactly as sent. Redaction applies to what is written
down, never to what is served.

### What is recorded, and what is not

**Inbound HTTP**, by decision — in-process middleware would require the project
to import x3 and tie the capability to one language. Connection headers are
neither forwarded nor recorded, so replaying cannot send a proxy's own settings
on to the application. When the target does not answer the client is told so with
`502` and **no line is written**: that answer came from the proxy, not from the
application.

## `x3 replay`

**What it catches:** a behavior change. `record` writes a run down; this sends it
again and compares. The behavior test is not a file somebody wrote — it is a
recording the machine took.

```
x3 replay [-config <file>] -target <url> -ledger <file> [-out <file>]
```

Three things are compared: the **status code**, the **headers named in
configuration**, and the **body, field by field**. Every field not named in a
rule is compared exactly — fail-closed, the direction every other gate points.

```
DIFF 2 res.body.state: value_differs
        recorded: created
        received: queued
```

### What is allowed to differ

A recording that compares timestamps fails on the second run.

```json
{ "replay": { "headers": ["Content-Type"], "normalize": [
    { "path": "res.body.created_at", "as": "time" },
    { "path": "res.body.id",         "as": "uuid" },
    { "path": "res.body.items.*.n",  "as": "number" },
    { "path": "res.headers.Date",    "as": "any" } ] } }
```

`time`, `uuid`, `number` and `any` are the four kinds. A normalized field is not
compared by value — but its **presence and kind still are**, so dropping it or
returning a string where a time was recorded is `kind_differs`. `headers`
defaults to `Content-Type`, because the shape of a body is behavior while `Date`
and `Content-Length` are not. This is the one place the comparison is opt-in
rather than fail-closed: a full header comparison is red on every run, and a gate
that is always red is a gate somebody switches off.

### Exemptions, and the dead ones

```json
{ "replay": { "ignore": [ { "path": "res.headers.X-Request-Id",
                            "reason": "per-request id, not behavior" } ] } }
```

Declared in `x3.json`, never inside the ledger, always with a reason — and an
exemption that silenced nothing is `dead_exemption` and **red**. `normalize`
rules are not held to this: a rule for a field that did not appear says nothing
about whether it is still needed.

### Carrying a session

Recording masks credentials, so a suite behind a login cannot simply be sent
again — the `Authorization` header on file says `<redacted:credential-header>`.
The way out is not to unmask the recording but to take a **fresh** value from the
run itself:

```json
{ "replay": { "carry": [ { "from": "res.body.token",
                           "into": "req.headers.Authorization",
                           "as": "Bearer {value}" } ] } }
```

Every answer is read for `from`, and whatever it yields is poured into `into` on
the requests that follow. A value may also come from the environment
(`"from": "env:X3_TOKEN"`). A value is carried **into a request only** — writing
into a response would mean editing the thing being compared. The template must
contain `{value}`. If the field is not in the recorded request at all it is
added, which is the common case: the header was masked away, and what replaces it
is a live value.

### Replaying in parallel

```json
{ "replay": { "workers": 8 } }
```

Against a target that waits 20 ms per call, 60 interactions: sequential 1.28 s,
8 workers **0.21 s**. Against a local application answering instantly the two are
the same, because what parallelism buys is the waiting, not the work. The report
is identical either way — findings are collected in interaction order.

Parallelism is declared, never assumed: only the project knows whether its
interactions are independent. **`workers` and `carry` together are a
configuration error**, refused before the run — a carried session needs the
recorded order, and going faster while getting a different answer is not going
faster.

### A database of its own

`testdb` and `replay` need no new feature to pair:

```
x3 testdb run -- ./start-app-and-replay.sh
```

`testdb run` clones a template database, exports its DSN, runs the command and
drops the database afterwards. What the engine deliberately does not do is start
the application itself — it does not know how, and a wrong guess would be worse
than the two lines of script.

### What a recording cannot send back

A masked value is not sent to the application: `<redacted:credential-header>` as
an `Authorization` header would come back `401`, and a reader would file an
identity error as a behavior change. Those values are counted instead:

```
x3 replay: 2 interaction(s) - 0 difference(s) - 0 exempted - 1 value(s) could not be sent back
```

The report carries no timestamp, and recorded and received values are truncated
and run through the `secrets` pattern set before they are printed — the report
that finds a leak must not become one.

### Calls the application makes

`x3 record` sees what the world asks of the application. This sees what the
application asks of the world — the rate service, the mail gateway, the payment
provider — and later answers those calls itself, so a replay does not reach
anybody outside.

```
x3 outbound record -listen :9101 -ledger out.jsonl
x3 outbound serve  -listen :9101 -ledger out.jsonl
```

This one is a **forward** proxy: the application is told about it the way every
HTTP client already understands, with `HTTP_PROXY`. In `record` mode the call
goes out and is written down with the same redaction. In `serve` mode nothing
goes out at all — the answer comes from the ledger, matched on method plus
scheme, host and path, in recorded order, so an application calling the same
endpoint twice gets the first answer first.

A call the ledger never saw is refused with `502` and counted, and the command
exits `1` when the count is above zero: during a replay a *new* outbound call is
new behavior, and a proxy that quietly let it through would hide exactly what the
replay is for. **Encrypted calls are refused, not tunnelled** — a `CONNECT` gets
`501`, because recording HTTPS would mean terminating TLS with a certificate of
x3's own, and believing you recorded a call you did not is worse than knowing you
did not.

## `x3 guard`

**What it catches:** a long run started against the wrong live environment. The
guards run first, and the command after `--` starts only if they pass — one
process, one decision, no wrapper script.

```
x3 guard [-config <file>] [-report <file>] [-only <tags>] [-skip <tags>] [-stamp] -- <command> [args...]
```

`-report` is the only way a report is written: **stdout belongs to the launched
command**. The command is started with x3's **own environment and working
directory** — nothing added, removed or rewritten — and its exit code is
returned verbatim, so a launched test run behaves exactly as it would without
the guard in front of it.

### The decision rule

| Guards | Decision | What happens |
|---|---|---|
| all pass | `launch` | the command runs; x3 exits with its exit code |
| red, all of them `policy: warn` | `launch` | the command runs; each red is printed as `WARN` first |
| at least one red with `policy: block` | `blocked` | **the command is never started**; x3 exits `1` |

A guard that could not run at all — missing variable, unreachable host, unknown
driver — counts as red. **A live guard whose answer is unknown is not an
answer**, and the switch is fail-closed.

### Choosing which guards run

One file usually holds every guard a project has, but the gate that starts a
worker has no business waiting on a guard belonging to a different binary.
`tags` plus `-only` / `-skip` pick a subset **out of the same file**, so a
narrower run is still the file everybody reviews rather than a second copy that
drifts.

```json
{ "name": "database-reachable", "kind": "sql", "tags": ["db", "slow"],
  "dsnEnv": "APP_DSN", "query": "select 1", "equals": "1" }
```

| Written | What runs |
|---|---|
| neither flag | **every guard in the file** |
| `-only a,b` | guards carrying `a` or `b`, **plus every guard with no tags at all** |
| `-skip a` | everything except the guards carrying `a` |
| both | `-skip` wins on a guard that matches both |

**A guard with no tags always runs**: narrowing a set must not drop the check
nobody got round to classifying. Three selections are refused outright with exit
`2`, before any guard runs — a tag no guard carries (a misspelled `-skip` would
otherwise skip nothing and read as if it had), a selection that leaves no guard,
and an empty tag. Whatever a selection dropped is named in the report's
`skipped` list and in the stderr summary: **a check that did not run must never
look like a check that passed.**

### Exit codes

| Code | Meaning |
|---|---|
| the command's own | the guards allowed the launch |
| `1` | a `block` guard was red, and the command was never started |
| `2` | the configuration or report could not be read or written, or the command could not start |

`1` carries two meanings — "blocked" and "the command itself exited 1". The
report separates them: `decision` is `blocked` in the first case, and `launch`
with an `exit` field in the second.

## Live guards in `x3.json`

Guards are **declared, not coded**. There is no Go file per guard and no plugin:
the engine knows three general source kinds — `sql`, `http`, `exec` — plus
`steps`, and everything project-specific is data.

Validation inside `live` is **strict and up front**: an unknown key, a key
belonging to a different kind, a missing required key, a duplicate name, an
unknown policy or an empty guard list stops the run *before any guard executes*.

**Two surfaces, one law.** `//x3:live` in source code is a *marker* — this code
talks to a real provider. The `live` section is where runnable guards are
*defined*. Both are declared in a dictionary inside the engine, and in both an
entry the dictionary does not know turns the run red.

### Fields every guard has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `kind` | yes | `sql`, `http`, `exec` or `steps` |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `tags` | no | what `-only` and `-skip` select on; a guard with none always runs |
| `timeoutMs` | no | defaults to `10000` (`steps`: `600000`); a dead dependency must not hang the gate forever |

### `kind: "sql"`

| Field | Required | Meaning |
|---|---|---|
| `dsnEnv` | yes | **name** of the variable holding the DSN; the DSN never appears in the file |
| `query` | yes | its first row, first column is the observed value |
| `driver` | no | defaults to `pgx`; a name this binary has not registered is a configuration error (exit `2`) |
| `equals` / `contains` | one of them | what the observed value must be |

An expectation is mandatory here: a query with no expectation asserts nothing,
because it is answered by an empty table.

```json
{ "name": "schema-current", "kind": "sql", "policy": "block",
  "dsnEnv": "APP_DATABASE_URL",
  "query": "select max(version)::text from schema_migrations", "equals": "0117" }
```

### `kind: "http"`

| Field | Required | Meaning |
|---|---|---|
| `url` | yes | the address; the request is always a `GET` |
| `status` | yes | the expected status code |
| `headerEnv` | no | header name → **name** of the variable holding its value |
| `jsonPath` | no | an RFC 6901 JSON Pointer into the body; without it the whole body is the value |
| `equals` / `contains` | no | with only `status`, the status code alone is the assertion |

```json
{ "name": "provider-agent-enabled", "kind": "http", "policy": "warn",
  "url": "https://api.provider.example/v1/agents/self", "status": 200,
  "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
  "jsonPath": "/agent/permissions/0", "equals": "outbound" }
```

### `kind: "exec"`

| Field | Required | Meaning |
|---|---|---|
| `command` | yes | executable to run |
| `args` | no | its arguments |
| `equals` / `contains` | no | what its trimmed stdout must be; without either, **exit code 0** is the assertion |

### `kind: "steps"` — a trial, not a reading

**What it catches:** a question that is not a fact you can read but an
experiment — *"does this module still compile once the application is removed
from the tree?"* Copy the tree, take the application out, build what is left,
put everything back. Without a multi-step kind the only way to write that is a
script inside the project, and a script is what x3 exists to remove: reviewed by
nobody, drifting when a path moves, never measured for whether it can still turn
red.

| Field | Required | Meaning |
|---|---|---|
| `steps` | yes | run **in order**; the first that does not hold ends the trial and names itself |
| `workspace` | no | a temporary working area: `copy` (required within it), `remove`, `write` |
| `equals` / `contains` | no | what the **last** step's output must be; without either, every step holding is the assertion |

A step takes `name`, `command`, `args`, `dir`, `env` (added to the inherited
environment for that step only), `output` (the same `must` / `mustNot` / `retry`
expectation the [work list](#the-criteria) writes, read against both streams) and
`timeoutMs`. Without an `output`, a step's assertion is its **exit code**.

```json
{ "name": "the core still builds once the application is removed",
  "kind": "steps", "policy": "block",
  "workspace": { "copy": ["go.mod", "go.sum", "cmd", "internal", "core"],
                 "remove": ["internal/app"] },
  "steps": [
    { "name": "the core compiles with no application in the tree",
      "dir": "core", "command": "go", "args": ["build", "./..."],
      "env": { "GOWORK": "off" } },
    { "name": "the binary names a released core, not the working copy",
      "command": "go", "args": ["version", "-m", "out/service"],
      "output": { "must": ["example.com/core v0."], "mustNot": ["(devel)"] } } ] }
```

The copy is taken from the **working tree**, not from the last commit: if an
uncommitted change crossed the boundary, the trial should see it in the same
run. Three things are refused rather than run, each closing a way to a silent
green: a trial with **no steps** (it would pass every time), a `copy` path that
is **not on disk** (a build that fell because a source was missing is red for
the wrong reason), and a `remove` path that is **not there** (a trial measuring
an absence it never created is green by construction).

Absolute paths and `..` are refused everywhere in a trial — a `remove` that
climbed out of the copy would delete from the working tree, and no gate may
damage the thing it measures. **The working area is removed in every case**,
including when a step fails mid-way, and the paths inside it are stripped out of
the output that reaches the report: the person reading the red opens the file
**in the repository**, not a copy that no longer exists.

`write` is what makes the control experiment possible from the configuration
alone: the same trial with one file written into the copy has to go red, and a
trial whose red has never been seen is not a trial.

### Secrets never enter the report

Credentials are referenced **by environment variable name only**. Before
anything is written, those values are stripped out of the observed value **and
out of the error text** — driver errors routinely quote the DSN they failed on,
and that string is replaced with `[redacted]`. An empty variable is an error,
not an empty credential: the guard goes red instead of asking anonymously and
reporting a misleading `401`. `TestSecretNeverLeaves` makes a fake driver fail
with the DSN inside its own error message and asserts the password is nowhere in
the marshalled result.

## The guard report

```json
{ "version": 1, "config": "…/guard-block-red.json",
  "guards": [
    { "name": "impossible-platform", "kind": "exec", "policy": "block",
      "status": "fail", "expected": "equals \"there-is-no-such-platform\"",
      "observed": "windows", "detail": "observed value is not equal to the expected value" } ],
  "summary": { "pass": 1, "warned": 0, "blocked": 1 },
  "decision": "blocked", "command": ["x3", "scan", "internal"] }
```

| Field | Notes |
|---|---|
| `guards[].status` | `pass`, `fail` (it ran and disagreed) or `error` (it could not run); both non-`pass` values are red |
| `guards[].policy` | the policy applied to **this** guard — always present, so the report explains its own decision |
| `expected` / `observed` / `detail` | what was wanted, what was seen, why it was red; secrets already redacted |
| `summary` | `pass` + `warned` (red under `warn`) + `blocked` (red under `block`) |
| `skipped` | the guards a selection left out, by name; absent when nothing was dropped |
| `decision` | `launch` or `blocked` |
| `exit` | the command's exit code. **Absent when `decision` is `blocked`** — that absence is the proof the command never ran |
| `startedAt` | present **only** with `-stamp` |

**No timestamp unless you ask for one**: the same configuration and the same
answers must produce the same bytes.

## `x3 guard:effective`

**What it catches:** a setting whose two lives have drifted apart. One is the
**record** — a row in a table, a field in a remote endpoint. The other is what is
**in force** — the value the running process actually loaded, and the value the
provider actually applies. They drift quietly, because every side is internally
consistent.

```
x3 guard:effective [-config <file>] [-out <file>] [-stamp]
```

Unlike `x3 guard` this launches nothing, so stdout is free for the report.

| Checks | Exit | What it means |
|---|---|---|
| every source agrees | `0` | the setting on paper is the setting in force |
| divergent, all `policy: warn` | `0` | printed as `WARN`, the run is not stopped |
| at least one divergent `policy: block` | `1` | the record and the world disagree |
| a source could not be read at all | as above | **red** — an unknown answer is not an answer |

## Effective checks in `x3.json`

Built out of the same three source kinds; what changes is the *role* a source
plays — one is the record, the rest are the world.

```json
{ "effective": { "checks": [
    { "name": "assistant-model", "policy": "block",
      "attempts": 3, "retryDelayMs": 500,
      "recorded": { "label": "database", "kind": "sql", "dsnEnv": "APP_DSN",
                    "query": "select model from settings where id = 1" },
      "effective": [
        { "label": "process", "kind": "http", "url": "${APP_BASE}/internal/settings",
          "status": 200, "jsonPath": "/model" },
        { "label": "provider", "kind": "http", "urlEnv": "PROVIDER_SETTINGS_URL",
          "status": 200, "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
          "jsonPath": "/model", "map": { "engine-2-2026-01-31": "engine-2" } } ] } ] } }
```

Validation is strict and up front, the same fail-closed rules the `live` section
has.

### Fields a check has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `attempts` | no | retries while the comparison disagrees; defaults to `1` |
| `retryDelayMs` | no | wait between attempts; defaults to `250` |
| `recorded` | yes | one reading: the setting as it was written down |
| `effective` | yes | one or more readings: the setting as it is in force; all must equal `recorded` |

**Retries exist because the world lags the record** — a process reloads a moment
after the row changes. One attempt is the default precisely so a retry is a
deliberate statement about how long the lag may be, never a way to wait out a
red.

### Fields a reading has

A reading is a `sql`, `http` or `exec` source, and every field documented under
[Live guards in `x3.json`](#live-guards-in-x3json) applies unchanged. Two are
added and two are **not allowed**:

| Field | Meaning |
|---|---|
| `label` | the name this source carries in the report |
| `map` | value mapping applied before the comparison; a value the map does not mention is compared as it came |
| ~~`equals`~~ / ~~`contains`~~ | **rejected here** — a reading has no expectation of its own; its expectation is the other readings |

`map` is what makes two spellings of the same setting comparable. A value the
map does not cover is *not* an error — it goes into the comparison unchanged, so
an incomplete mapping produces an explainable red, never a false green, and the
report keeps the raw value next to the mapped one.

## The effective report

```json
{ "version": 1, "checks": [
    { "name": "platform", "policy": "block", "status": "fail", "attempts": 1,
      "recorded": { "label": "record", "kind": "exec", "value": "windows" },
      "effective": [ { "label": "world", "kind": "exec", "value": "amd64" } ],
      "detail": "record says \"windows\", but world says \"amd64\"" } ],
  "summary": { "pass": 0, "warned": 0, "blocked": 1 } }
```

`status` is `pass`, `fail` (the sources disagreed) or `error` (a source could not
be read); both non-`pass` values are red. `attempts` says how many the answer
actually needed — a `2` here says the world was late, not wrong. Each reading
carries its `label`, `kind`, compared `value`, the `raw` value when a mapping
changed it, and `error` when it could not be read. `startedAt` appears **only**
with `-stamp`.

Secrets follow the guards' law, and `TestEffectiveSecretNeverLeaves` holds it
for this report specifically.

## `x3 adoption`

**What it catches:** an engine half used and nobody noticing — a capability no
gate script runs, a settings section written but empty, a checksum vouching for
a binary the version gate already refuses, and a pile of test files the inline
examples were supposed to replace.

```
x3 adoption [-config <file>] [-out <file>] [dir]
```

**It is a mirror, not a fence.** Findings default to `warn`, so the run stays
green and stops nobody: no project has to use every command, but a project that
is not using one should be able to see that. `"policy": "block"` gives it teeth.

**The list is the engine's own.** The commands measured are the entries of the
binary's dispatch table, read at run time. A copy of that list kept beside the
project would go stale the day a command is added, and a stale list reports its
own blindness as full coverage — the measure fails exactly where it is needed.

```json
{ "adoption": {
    "runners": ["check.ps1"],
    "invoke": ["[$]bin ['\"]?([a-z]+(?::[a-z]+)?)"],
    "tests": ["**/*_test.go"],
    "token": 2, "split": 300,
    "exempt": { "lang": "one language only; there is no prose to gate" },
    "policy": "warn" } }
```

| Field | Meaning |
|---|---|
| `runners` | the gate scripts that call the engine; declared, not discovered |
| `invoke` | how this project spells a call — one capture group, the command name |
| `sources` / `exclude` | where `//x3:` directives are counted (default `**/*.go`) |
| `tests` | the files whose number is supposed to be falling |
| `token` | the ceiling under which a section is written rather than working |
| `split` | the line count past which a configuration wants `include` |
| `exempt` | command → **reason**; a reason is required and a dead one is a finding |
| `policy` | `warn` (default) or `block` |

### How the engine is called

The engine's name cannot be assumed. x3's own gate compiles a binary to a
run-scoped path and calls it through a variable — `& $bin scan .` — and a
measure that only knew `x3 scan` would report *"this project runs nothing"* on
the very repository that runs everything. So the spelling is declared, and the
default (`x3 <command>`) is only a default. A captured word is kept **only if
the engine has a command by that name**, which is what keeps the sentence "the
x3 binary is missing" in a comment from counting as a run — and comments are
stripped before the pattern is applied, because a step that was deleted must
not stay alive in the prose that described it.

### Green

```
  COMMANDS     [##################..]  20/22 run, 2 exempt
  SECTIONS     1 in the configuration, 0 barely in force
  TEST FILES   0 standing against 0 inline example(s)

  EXEMPT, WITH A REASON (2): lang - version

x3 adoption: 22 command(s) - 1 section(s) - 0 block, 0 warn
```

### Red

```
BLOCK docs section_token
        puts 2 name(s) in force, the token ceiling is 2; the section is
        written, not working
BLOCK tests_remain
        1 test file(s) still stand against 0 inline example(s); the engine's
        claim is that the second replaces the first
BLOCK dead_pin
        update.pin carries 1 checksum(s) below min_version v0.10.0 (v0.9.0);
        the version gate already refuses those binaries, so the record vouches
        for nobody
```

**"The section exists" is not a measure.** A section putting one rule in force
and a section putting thirty in force would otherwise read the same. Depth is
counted the way the engine counts it — `policy: "warn"` is not in force, here or
in a consistency rule — and it is only asked of a section that actually holds a
list: a ceiling written as a number has no depth, and calling it empty would be
a finding about nothing.

**An exemption carries a reason, and a dead one speaks.** A command exempted and
then actually run is a finding; a name exempted that the engine does not have
stops the run with exit `2`, because an exemption for nothing hides the day the
name changed. The sections describing the engine's own workings — `x3`,
`update`, `baseline`, `cache`, and `adoption` itself — are counted in neither
direction: they put no check in force.

### Findings

| Code | Meaning |
|---|---|
| `command_unused` | no runner calls it and no exemption says why not |
| `section_token` | a section holding a list puts `token` or fewer names in force |
| `dead_exemption` | exempted, and run anyway |
| `tests_remain` | test files still stand where inline examples were meant to be |
| `dead_pin` | `update.pin` vouches for a release below `x3.min_version` |
| `config_one_file` | the configuration passed `split` lines and declares no `include` |

## `x3 version`

```
x3 version
```

Prints the release tag embedded at build time and exits `0` — one line, nothing
else, so a gate can compare it without parsing. A binary not produced by a
release run prints `unreleased`; an untagged binary is not a published one, and
a gate that pins versions should treat it as red.

## `x3 update`

**What it catches:** nothing — it removes the downloader every consuming project
would otherwise write for itself. The binary replaces itself with a published
one, after verifying its SHA256.

```
x3 update [-config <file>] [-version <tag>] [-source <address|dir|owner/repo>] [-check]
```

The order is fixed: resolve the tag (`-version`, else the release pointer); find
the checksum this platform's binary must have (from
[`update.pin`](#updatepin--the-checksum-the-project-itself-vouches-for) when the
project wrote one, else the release's `SHA256SUMS.txt`); download; compute and
compare; put it in place.

**A sum that does not match stops before step 5 and the running binary is left
exactly as it was**, as is a release listing no binary for this platform — both
exit `1` and name the file left in place. Not being able to *reach* the source
exits `2`: "the release refused me" and "the network refused me" are not the
same event and must not wear the same colour.

**The layout.** Every source, remote or local, is read as `<source>/<ref>/<file>`:
`main/LATEST` (one line, the newest tag), `<tag>/SHA256SUMS.txt`, and
`<tag>/x3-<goos>-<goarch>[.exe]`. `LATEST` is written by the same run that builds
the binaries — a separate step would drift, and a pointer naming a release nobody
published is the quietest way to break an update.

**Where it downloads from**, first answer wins: `-source`, `X3_UPDATE_SOURCE`,
`update.source`, then the engine's own public repository. Outside in, on
purpose: pointing one run at a mirror should not require editing a tracked file.
A source may be an address, an `owner/repository` shorthand, or **a local
directory** — the directory case is not a test fixture but how an air-gapped or
mirrored environment publishes the same three files onto a share. An existing
directory wins over the shorthand.

**`-check` changes nothing**: it prints the published tag and exits `1` if that
tag is newer than the running binary. The tag goes to stdout on a line of its
own; everything written for a human goes to stderr.

**Replacing a running file.** The new bytes are written next to the target
first, because a rename is only atomic within one filesystem. On Windows a
running executable cannot be overwritten but can be renamed, so the sequence is
write, move the old one aside, put the new one in place — and if the last step
fails the old name is given back.

```json
{ "update": { "source": "owner/repository", "latestRef": "main", "timeoutMs": 120000 } }
```

An unknown key in it is an error, not a silent skip.

### `update.pin` — the checksum the project itself vouches for

`SHA256SUMS.txt` ships **inside the release it describes**. It catches a
truncated download, a mirror that fell behind, a corrupted file. It cannot catch
a compromised release: whoever can replace the binary can replace the list
beside it. **A release that vouches for itself is not a supply chain guarantee.**

```json
{ "update": { "pin": { "v0.33.0": {
      "x3-windows-amd64.exe": "e2c2bd46...",
      "x3-linux-amd64":       "b21f4cc9..." } } } }
```

**When a pin is written, `SHA256SUMS.txt` is not read at all.** The stderr line
says which authority it obeyed, so a run never leaves that ambiguous. The pin is
keyed by **release tag**, not by binary name alone, which is what lets the engine
tell two refusals apart:

| Situation | Result |
|---|---|
| the tag is pinned and the bytes match | installed |
| the tag is pinned and the bytes differ | exit `1`, binary left in place, message names `update.pin` |
| the tag is not in `pin` | exit `1`, naming the tags that *are* pinned |
| the tag is pinned but not for this platform | exit `1` — a machine nobody pinned gets no weaker guarantee |
| no `pin` at all | `SHA256SUMS.txt` decides, exactly as before |

Because an unpinned tag is refused, a pinned project does not follow `LATEST` by
accident: a new release enters the project the day somebody writes its checksum
down. A malformed pin is a **configuration error** (exit `2`) rather than a
mismatch — reported as a mismatch, a mistyped checksum would leave a project
unable to update and unable to see why.

### The minimum version gate

```json
{ "x3": { "min_version": "v0.30.0" } }
```

This runs **before every command**:

```
x3: RED - this binary is v0.29.0, the project requires v0.30.0 or newer
	run: x3 update
```

`x3 update` is the one command exempt — it is the answer the gate points at, and
a red with no way out is a wall, not a gate. **An untagged binary satisfies
nothing.** A `git describe` suffix is ignored (`v0.30.0-3-gabc1234` counts as
`v0.30.0`, since those commits come after the tag). A requirement that is written
must parse: no file and no section means no requirement, but a value that is not
a release tag is an error — a misspelled requirement silently ignored leaves its
author believing a gate is running.

## `x3 testdb`

**What it catches:** tests that serialise on one shared database, and tests that
pay for the migrations every time. `x3 testdb` gives a run its own PostgreSQL
database by **cloning a prepared template**, hands the command a DSN through the
environment, and drops the database when the command is done.

```
x3 testdb create [-config <file>]
x3 testdb drop   [-config <file>] (-name <database> | -stale)
x3 testdb list   [-config <file>] [-stale]
x3 testdb run    [-config <file>] [-keep] -- <command> [args...]
```

`create` prints its **DSN on stdout**, one line and nothing else, so a shell can
capture it. `run` is the shape most projects want: one process, a fresh
database, automatic cleanup even when the command fails; `-keep` leaves it
behind, which is exactly what `drop -stale` later collects.

### Two invariants

**Speed.** Measured on PostgreSQL 18.2 over loopback, five runs of a 40-table,
40-index schema: **238–263 ms** to clone the template against **275–292 ms** to
create an empty database and replay the same DDL; end to end `x3 testdb create`
took **409 ms**. The gap widens with the schema — cloning is one directory copy
whatever the migration count.

**Safety.** Every name this command touches has to be one x3 made, checked
before a byte reaches the server: the name matches `^[a-z_][a-z0-9_]{0,62}$`
(`CREATE DATABASE` takes no bound parameters, so this is the injection gate, not
a style rule), and it carries the configured `prefix` **and** the creation stamp
x3 writes into it. A name that fails either rule exits **1** — the gate refused
it — while a name that passes and then cannot be reached exits **2**. That
difference is what makes the gate observable from outside.

### `testdb` in `x3.json`

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "template": "app_test_template",
    "prefix": "apptest_", "dsnEnv": "APP_TEST_DSN", "maxAgeMinutes": 120,
    "migrate": { "command": "./migrate", "args": ["up"], "timeoutMs": 60000 } } }
```

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the variable holding the maintenance DSN. Point it at a maintenance database, never at the template: a template with an open connection cannot be cloned |
| `driver` | no | defaults to `pgx`; an unregistered name is a configuration error (exit `2`) |
| `prefix` | no | defaults to `x3test_`, and it is the **authority boundary** — nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | without it an empty database is created and the migration hook does the work |
| `dsnEnv` | no | the variable the new DSN is exported as; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover is stale; defaults to `120` |
| `migrate` | no | `command`, `args`, `timeoutMs`, run after creation with the DSN in the environment |

The DSN handed to the command is the maintenance DSN with **only the database
name changed**, so credentials and options carry over; both PostgreSQL spellings
are understood. **The creation time is in the name** — PostgreSQL does not record
it — which is what lets `-stale` work on any server with no extra table and no
privileges, and it is also why a database x3 did not name has no age and is
never touched. **If the migration hook fails, the database is dropped**: a
half-built schema is worse than none.

### Secrets and errors

The maintenance DSN is named by environment variable only, and its value is
stripped out of every error message before it is printed. The DSN of the
*created* database is deliberately printed by `create` — that is the point of the
subcommand — but under `run` it is never printed, only passed through the
environment.

## Speed

Files are independent, so reading and parsing them is done on every core and the
results are put back in file order — the report is byte-identical whatever the
core count. Measured on a real Go application of **1174 Go files** (2040 in
total), sixteen cores:

| Command | Before | After |
|---|---|---|
| `x3 lang` | 5.09 s | **0.24 s** |
| `x3 scan` | — | **0.12 s** |
| `x3 secrets` (all 2040 files) | — | **0.54 s** |

The same run produced the same bytes before and after, which is the part worth
checking: a parallel walk that reordered its findings would turn every later
comparison into noise.

### The incremental cache

A run can remember what it measured, keyed on the **content** of each file.

```json
{ "cache": { "dir": ".x3cache" } }
```

A directory, not a file: the file name comes from the command, because two
commands sharing one file would each delete the other's entries. Nothing is
written unless the section is there, and the directory belongs in `.gitignore`.
`x3 scan`, `x3 lang` and `x3 secrets` read it; `-cache <file>` points one run
elsewhere and `-no-cache` measures everything again.

An entry is used only when three things match: the **engine version**, a
**fingerprint of the whole configuration**, and the file's **content hash**.
Guessing which section affects which checker would be cheaper and would
eventually be wrong.

| Command | Full scan | Cached | Cache size |
|---|---|---|---|
| `x3 secrets` (2040 files) | 0.34 s | **0.12 s** | 0.5 MB |
| `x3 scan` (1174 Go files) | 0.13 s | **0.09 s** | 145 KB |
| `x3 lang` (1174 Go files) | 0.17 s | 0.18 s | 5.8 MB |

`lang` is the honest row: on that application the gate is red on thousands of
lines, and reading 5.8 MB of stored findings costs as much as parsing the files
again. **The cache pays off when a checker's output is much smaller than its
input** — which is why it is declared per project rather than switched on for
everybody. Every one of those runs produced a report byte-identical to the
uncached one.

## Files the engine reads back

Some of what a gate reads is not source but state the project keeps beside it:
the frozen baselines, a findings baseline, the open-work list, the cache. They
are read through one reader — the same one that reads `x3.json` — and it **drops
a leading byte order mark**. Windows tools write one while Go's JSON decoder
calls it an invalid character, so having the settings file forgive it and a
baseline refuse it meant two files written by the same editor behaved
differently, and the error named a character nobody typed.

## Splitting the configuration

One `x3.json` is enough for a small repository and wrong for a large one: the
rules that belong to a module end up far from the module, so removing the module
leaves its rules behind, guarding nothing.

```json
{ "include": ["x3/*.json", "apps/*/x3.json"], "language": { "allowed": "en" } }
```

Every command reads the merged result, under one merge law that knows nothing
about any section's schema:

| Both sides are | Result |
|---|---|
| lists | the parts are **added**, root first, then the files in name order |
| objects | merged key by key, recursively |
| anything else | **refused** — the run stops and names both files and the key |

Nothing is silently overwritten: a setting that quietly loses to another file is
a setting whose author believes it is in force. Four more refusals, all
fail-closed: **a pattern that matches no file** (an `include` that does not work
is a set of rules nobody notices is missing); **a part that includes** (parts are
one level deep, so the whole configuration is readable from the root);
**discovery** (parts are declared, never found by scanning — a file dropped into
a folder must not add a rule nobody reviewed); and **a missing section**, still
an error for the command that needs it. Ordering is by file name, so the merged
configuration is the same on every run and machine. A project that does not
split pays nothing.

## Pilot: a real `x3.json`

x3 is piloted inside a real production application. Nothing about that
application is encoded in the engine; what follows is its configuration file,
with generic names, as an example of what live guards are actually for. The
pilot's problem is the one every deployment has: a long test or migration run
that starts against a **wrong live environment** wastes an hour and can corrupt
state.

```json
{ "live": { "guards": [
    { "name": "schema-current", "kind": "sql", "policy": "block",
      "dsnEnv": "APP_DATABASE_URL",
      "query": "select max(version)::text from schema_migrations", "equals": "0117" },
    { "name": "catalog-engine-address", "kind": "sql", "policy": "block",
      "dsnEnv": "APP_DATABASE_URL",
      "query": "select engine_ref from capability_catalog where tier = 'standard'",
      "equals": "provider:engine-v3" },
    { "name": "provider-agent-permission", "kind": "http", "policy": "warn",
      "url": "https://api.provider.example/v1/agents/self", "status": 200,
      "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
      "jsonPath": "/agent/permissions/0", "equals": "outbound" } ] } }
```

| Guard | The question | Why that policy |
|---|---|---|
| `schema-current` | is the migration ledger at the schema version this code expects? | `block` — an older schema produces failures that look like code bugs and are not |
| `catalog-engine-address` | does the catalog row for this tier still point at the engine the run assumes? | `block` — a stale row silently routes the whole run somewhere else |
| `provider-agent-permission` | does the provider still grant this agent the permission the run needs? | `warn` — an external provider having a bad minute should not stop local work, but nobody should discover it an hour in |

The gate is then one line, with no shell logic deciding anything:

```
x3 guard -config x3.json -report build/guards.json -- go test ./...
```

**The red that made this worth building.** When the token holds a rotated key the
endpoint answers `401`, and the run says so before anything starts. The token
itself appears nowhere — not in the config, not on stderr, not in the report.
Change that guard's policy to `block` and the same situation stops the run
instead of warning about it; that one word is the whole difference.

## Releases and reproducible builds

The engine is published as binaries into a public repository carrying nothing
else: two binaries, `SHA256SUMS.txt` and a generated `README.md`. The source
repository is private, so the binary and its documentation are the whole public
surface.

One command produces a release, and if any step fails nothing is published:

1. it builds `windows/amd64` and `linux/amd64` with
   `-trimpath -buildvcs=false -ldflags "-s -w -buildid= -X main.version=<tag>"`
   and `CGO_ENABLED=0`, so the binary carries no build path, no build id and no
   VCS stamp — only the tag;
2. it builds **each target a second time** and compares the two SHA256s. A build
   that does not reproduce is not published, and that comparison happens on every
   release rather than in a one-off experiment;
3. it writes `SHA256SUMS.txt` and generates the public `README.md` from this
   document plus a template, stamping the tag and the SHA256 of both sources into
   the generated file;
4. it re-reads what it just wrote and runs the staleness gate against it.

**The staleness gate** recomputes the SHA256 of this document and of the template
and compares them with the stamp in the published README. Either changing after
the last release run turns the step **red**: the binaries do one thing and the
README describes another. A publish directory that is not configured, or
configured and missing, is red as well and says `NOT GENERATED` — deliberately
not a green skip, because "nobody has published yet" and "the publication is
current" are not the same answer. The step carries its own control experiment: it
asks the same question with a deliberately wrong document hash and requires a
red.

Two further gates stand between this document and the public repository. **The
leak gate** reads the generated README against a list of patterns the project
keeps privately — real module and directory names, project and customer names,
local paths, account names — and a single match stops the publication; the
finding names the pattern and masks the value, because a gate that printed what
it found would be a second leak. **The size gate** is the engine's own `freeze`
cap over the generated README: it takes no debt, cannot be lowered by `-update`,
and a document above it is red. Both run in `check.ps1` as well, each with a
two-way control experiment.

The publish directory is configuration and never a constant in the code:
`-DistDir` wins, then `X3_DIST_DIR`, then `dist.dir` in `x3.json`.

## Using x3 from another project

**Integration is by binary, not by import.** The consuming project does not add
x3 to its `go.mod`, does not use a `replace`, and does not put it in a `go.work`.
The directives are plain comments, so the consuming project's compiler never sees
them and its dependency graph never learns that x3 exists.

**Pin a version, and let the engine fetch itself.** The project writes the
version it requires into its own `x3.json`
([the minimum version gate](#the-minimum-version-gate)) and calls
[`x3 update`](#x3-update) to obtain that binary. Nothing else about x3 is
tracked: no downloader, no checksum file, no path.

This corrects earlier advice, and the reason is worth keeping. The first
integration had the project carry its own script to read a pinned version,
download the binary and verify the sum. That script was correct and still wrong:
every project using the engine would write the same one, each with its own bugs,
and the engine could fix none of them. A checked-in path is worse still — green
on the machine that wrote it, unmeasured everywhere else, and unable to say
*which* build ran.

**A missing or mismatched binary is red, not skipped.** The pilot's gate was
fail-open at first: no binary meant a warning and a normal start. That is the
failure this engine exists to prevent. **"The tool was not there" and "the tool
found nothing" must never produce the same colour.**

**The counts are the engine's job, not yours.** A scan of a tree with no
directives exits `0` and reports `0 red`, so a gate trusting the exit code alone
turns "delete the directives" into a way to go green. The first answer was to
have the project read the report and assert on it — which worked, and meant every
project wrote the same counter with its own bugs. The count now belongs to the
configuration: see [Expectations](#expectations).

**Directives arrive next to the existing tests, not instead of them.** Nothing is
migrated until its x3 equivalent has been seen to go red on a deliberately broken
input.

## Gaps we know about

Stated plainly, because a capabilities document that lists only strengths is a
sales page.

**Baselines and exclusions.** A baseline is coarser than the finding it holds —
`comments` identifies a finding by rule and file, so a file already owing one
over-long block can grow a second unseen. An exclusion is only as narrow as
somebody wrote it: nothing checks that an `ignore` is not swallowing a real
credential, only that it swallows *something*. Exclusions apply to the scan, not
to redaction. `of: files` counts one directory, not a subtree. Nothing checks
that a baseline was reviewed — `-update-baseline` refuses growth, but the first
write accepts whatever the tree owes that day.

**Expectations count directives, and only from `scan`.** They say a minimum,
never a maximum, and cannot say "these two exact directives".

**Updates verify a checksum, not a signature.** `update.pin` binds *bytes*, and
is worth exactly as much as the review of the commit that introduced the line;
nothing here checks a key. A pin has to be maintained by hand, and that friction
is the feature — it is still friction. The minimum version gate is enforced from
v0.30.0 on, so it protects you from binaries newer than the gate itself, not
from every old one.

**The cache is per file, not per project.** A checker whose answer depends on
more than one file at a time — `arch`, `freeze`, `docs`, `boxes` — does not use
it.

**Recording and replay.** Parallel replay is opt-in and all-or-nothing. A masked
value cannot be sent back, so a suite behind authentication replays as an
unauthenticated one unless a value is carried. Outbound recording is plain HTTP
only — `CONNECT` is refused rather than tunnelled. And a recording is only as
good as the traffic it saw.

**Every matcher reads shapes, not meaning.** `symbol` and `pairing` read names,
not types; `flow` does not follow a copy; `exposure` sees tags, not
serialization; `duplication` and `vocabulary` compare text and words, not
meaning; `containment` reads paths, not contents; `consistency` reads four
extractor shapes and no more; `secrets` reads formats; `suspect` finds shapes,
not lies; a lane is paths, not intent; an output expectation is a substring, not
an understanding; and `syntax` carries one parser of its own. A text file cannot
carry an exemption, and the inward `deps` form does not see a component's inside.

**`surface` reads syntax, not types.** The callers of a *member* cannot be
resolved without a type checker, so a changed field or method reports the files
that name its owning type and says so (`usersBasis: "type"`); a dot-import is
invisible either way. A constant's **value** is not frozen, only its declared
type — freeze a value set with `freeze` if the number is the contract. The
surface is the union across build constraints, so a platform-only symbol is in
it. And it measures the API a caller *writes*, never what a call *does*.

**`boxes` measures evidence, not completion.** A command criterion runs where the
gate runs, and a move is trusted once its target exists.

**Examples are one call, not a scenario.** An example cannot expect a panic and
cannot read a value it mutated.

**Guards compare strings.** `http` guards are `GET` only, guards run one after
another, and the guard report is written once, after the command finishes. The
`sql` and `http` kinds are control-tested in Go rather than in the gate script,
which covers `exec` only. Effective checks compare strings too, never remember,
and `map` is a lookup table, not a rule.

**`x3 testdb` speaks PostgreSQL only.** Nothing prevents two runs from sharing a
template, and it keeps no record of its own beyond what it encodes in a name.

**The language gate speaks one language** — `en` is the only embedded dictionary.

## Control experiments

**No gate here is trusted because it is green.** Every capability has a pair in
`check.ps1` — one tree it must pass and one it must fail — and the pair is the
evidence, because a green nobody has seen fail proves only that nothing ran. The
step names below are the ones the gate prints.

| Step | The pair, and what only the red half proves |
|---|---|
| `control experiment` | a well-formed sample `0`, a broken one `1` |
| `case control experiment` | an example that holds, one whose value is wrong, one with no payload, and one **nothing ran** — the last is why `never_ran` exists; plus an example using its file's imports `0`, and a tree with one broken example `1` **whose three sound neighbours still passed** |
| `language gate` | the repository `0`; a planted word `1`; a green tree with its allow list `0` **and without it `1`** — an allow list never seen to change an answer is decoration |
| `docs gate` | this repository `0`; a rule whose counterpart directory cannot exist `1` |
| `arch control experiment` | a green/red pair for every rule kind and every escape hatch: `absent` present, missing and dead; `skip` off, on and dead; `comments` read, exempt and embedded; `relativeTo` both ways; `minimum` met and short; `exclude` applying, dead and emptying the rule |
| `freeze control experiment` | the surface green, one name added red, and **`-update` on the grown tree red with the file unchanged** |
| `freeze count control experiment` | held, grown, shrunk and capped trees, each also under `-update`; the capped key stays out of the baseline **and the update itself exits `1`** |
| `surface control experiment` | six directions on one tree: **no baseline** (red, because growth is green here), recorded, untouched, a changed signature red **and naming its caller**, the same break allowed, a symbol *added* green, and the allow gone dead |
| `freeze scope control experiment` | one tree five ways; the exclusion written properly is `0` **on a tree built to be red without it**, and the same intent written as `"!…"` inside `sources` is `2` |
| `baseline control experiment` | no baseline, written, re-run, **the same debt moved down the file** (`0`), grown, `-update-baseline` refused, and one debt paid leaving `dead_baseline` |
| `secrets control experiment` | clean, leaky, exempted, a dead exemption, and this repository — the clean and exempted rows are what separate a gate from a noise generator |
| `secrets ignore control experiment` | a noisy tree with and without its exclusions, **then a real address added back**, then a dead exclusion, then a lookaround (`2`) |
| `comments control experiment` | inside the limit, one line over, exempted with a reason, an exemption that silences nothing |
| `boxes control experiment` | finished work left open, unfinished work closed, a condition unmet then met, two boxes closed with and without proof, and a `sql` criterion whose DSN is empty — counted, never green |
| `boxes document list control experiment` | the same for a document list, and **the same tree with one state declared `open` then `silent`** — one line of configuration decides, nothing else changes |
| `boxes criterion fidelity control experiment` | criteria that stopped measuring, each red with a control that removes the rule and returns the tree to green. Its fourth row is deliberately **green**: a test that was never written, measured by exit code alone, with no `output` |
| `syntax control experiment` | parsing, broken, a parser that is not installed (red), and the same check under `missing: "warn"` |
| `scope control experiment` | inside the lane, crossing it, crossing with a reason; then the branch form both ways, including a violation in the first commit under a clean one, and a closed lane |
| `test control experiment` | six directions on one tree, including **a full run when a file belongs to no unit** and a cache that answers, then measures again once the file changes |
| `record` / `replay control experiment` | a ledger whose credential header and planted key are hidden **while an ordinary field is still there**; then a replay without a `normalize` rule (red), with it (green), and against a drifted application (red) |
| `outbound control experiment` | a call recorded through the proxy, the same answer served **with the far side shut down**, and an unrecorded call refused `502` |
| `guard control experiment` | green, blocked and warned — the launched command proves it ran by writing a file, and the blocked row proves it did not |
| `guard selection control experiment` | one file, only the flags changing; a mistyped tag exits `2` rather than skipping nothing |
| `multi-step trial control experiment` | the same trial green, red once an import is *written* into the copy, red on an empty removal — and **zero working areas left behind, the reds included** |
| `effective control experiment` | agreement, divergence under `block`, the same divergence under `warn` |
| `update control experiment` | installed, a planted checksum refused, the version gate both ways, and **a pin disagreeing with a release whose own checksum list is perfect** — which is exactly how a compromised release looks |
| `testdb control experiment` | a foreign name refused at the gate (`1`) **and our own name reaching an unreachable server (`2`)** — a gate that refused every name would also exit `1` |
| `expect control experiment` | the count met, one guard short, the directives deleted, and the same tree with no expectation |
| `public leak gate` | the published documents clean, and a planted tree in which **every** forbidden pattern speaks |
| `public size gate` | the published README under the cap, and a document one line over it |
| `dist gate` | the publication current, and the same question asked with a deliberately wrong document hash |

Whatever cannot be arranged from a shell — a database, a network, a fake driver,
a mapping, a retry — is control-tested in Go instead, to the same rule: each
green is shown next to the red that proves it was measured.

## The documentation gate

This repository holds itself to the rule it ships: a change under `internal/` or
`cmd/` must carry a change under `docs/` in the same diff. The gate is
[`x3 docs`](#x3-docs) reading this repository's own `x3.json` — the same command
any project would run.

```
docs: none - <why the reader loses nothing>
```

A reasoned skip is written in the commit body; for the run before the commit
exists, pass the same line with `-reason`. The marker with nothing after it is
red, on purpose.

---

<!-- x3-dist version=v0.56.0 capabilities=4a97e6e2295f0869c5effc815a8978e1ed315dbeb3f3cfe8080d9e92a7ee7825 template=557480518c2d751cf2629e1c3bd9268eb986f84aae65f429d747ff0ba8daab65 -->
