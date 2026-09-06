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

**Current version: `v0.5.0`**

## Download

| File | Platform | Size | SHA256 |
|---|---|---|---|
| `x3-windows-amd64.exe` | windows/amd64 | 12.2 MB | `cf8bb3045f3e82fc99ca2cfb8c7ddd6d511a206fc06072fcedefb68c3addac5c` |
| `x3-linux-amd64` | linux/amd64 | 11.8 MB | `4ba55a7135d43a22b3388e3a70299191260bb86a81cec455dfe912cb1096569e` |

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
x3 guard -config x3.json -- go test ./...   # live checks, then the command
x3 guard:effective -config x3.json          # the setting on paper vs in force
x3 testdb run -config x3.json -- go test ./...   # a fresh database for this run
```

Exit codes are the same for every command: **0** green, **1** red, **2** usage
or I/O error. When `guard` launches the command, the command's own exit code is
returned instead.

Project configuration lives in one file, `x3.json`: the `language` section for
the language gate, the `arch` section for the architecture rules, the `live`
section for the guards, the `effective` section for the recorded-versus-in-force
comparisons, the `testdb` section for run-lifetime databases. A large repository
splits that file: the root declares its parts with `include`, lists are added
and objects merged, and anything else set twice stops the run. All of them are documented below, with the schema and a worked
example.

---

**What the engine does today** — one worked example per capability, quoted from
the control samples that live in the repository. The README describes the
vision; this file describes the build. If something is on the README roadmap
and not in this file, it does not exist yet.

> **Documentation gate.** A change under `internal/` or `cmd/` must carry a
> change under `docs/` in the same diff, or `check.ps1` turns red. See
> [The documentation gate](#the-documentation-gate) at the bottom.

## Contents

- [What is built and what is not](#what-is-built-and-what-is-not)
- [`x3 scan`](#x3-scan) — command, flags, exit codes, what it walks
- [Scopes](#scopes) — where you write a directive decides what it binds
- [The dictionary](#the-dictionary) — the six directive types
- [Error codes](#error-codes) — the four ways a directive turns red
- [The JSON report](#the-json-report)
- [`x3 lang`](#x3-lang) — the language gate: one language outside comments, dictionary in reverse
- [`x3 arch`](#x3-arch) — architecture rules: which component may import which
- [`x3 guard`](#x3-guard) — run live guards, then launch a command only if they pass
- [`x3 version`](#x3-version) — the release tag embedded in the binary
- [Live guards in `x3.json`](#live-guards-in-x3json) — the three source kinds and the warn/block switch
- [The guard report](#the-guard-report)
- [`x3 guard:effective`](#x3-guardeffective) — compare a setting as recorded with the same setting as it is actually in force
- [Effective checks in `x3.json`](#effective-checks-in-x3json) — the recorded side, the effective sides, mapping and retries
- [The effective report](#the-effective-report)
- [`x3 testdb`](#x3-testdb) — run-lifetime test databases: clone, migrate, drop, collect the leftovers
- [Splitting the configuration](#splitting-the-configuration) — one file, or many parts the root declares
- [Pilot: a real `x3.json`](#pilot-a-real-x3json)
- [Releases and reproducible builds](#releases-and-reproducible-builds)
- [Using x3 from another project](#using-x3-from-another-project)
- [Gaps we know about](#gaps-we-know-about)
- [The documentation gate](#the-documentation-gate)

## What is built and what is not

The architecture has four components (README, *Architecture*). One is
implemented:

| Component | State |
|---|---|
| **Scanner** (`internal/scan`) | **implemented** — walks the AST, collects directives, resolves scopes |
| **Dictionary** (`internal/scan`, `dict`) | **implemented** — six types, format and scope checks only |
| **Recorder** (`internal/engine`) | interface only — no implementation |
| **Ledger** (`internal/engine`) | interface only — no implementation |

Alongside them, one capability that is not part of that four-component picture:

| Capability | State |
|---|---|
| **Live guards** (`internal/live`) | **implemented** — `sql`, `http` and `exec` checks declared in `x3.json`, a `warn`/`block` policy each, and the `x3 guard` command that launches a command only when they allow it |
| **Language gate** (`internal/lang`) | **implemented** — `x3 lang` checks that everything outside comments is written in one language, against an embedded English dictionary plus the project's own `language.allow` list |
| **Effective checks** (`internal/live`) | **implemented** — `x3 guard:effective` reads one setting from the place it is *recorded* and from every place it is *in force*, and turns a divergence red |
| **Architecture rules** (`internal/arch`) | **implemented** — `x3 arch` compares the import graph against the components and rules a project declares in `x3.json`; one of the nine specified rule kinds (`deps` with `match: "import"`) has a verifier |
| **Test databases** (`internal/testdb`) | **implemented** — `x3 testdb` clones a template database per run, applies a migration hook, drops it when the command finishes, and collects what earlier runs left behind |

What is implemented is a **language check**, not a behaviour check. The scanner
answers three questions about every `//x3:` line it finds:

1. Is this directive type known — does it have a verifier at all?
2. Is its shape right — are the required sub-types and the reason present?
3. Is it in a scope where this type is legal?

It never calls your code, never runs a case, never proves that a `rule` holds.
A green `x3 scan` means *"your directives are well formed"*, nothing more.
(`x3 guard`, further down, *does* reach the outside world — but it checks the
environment a run is about to happen in, not the behaviour of your code.) That
distinction is deliberate: behaviour verification is the next stage, and
claiming it now would be a promise the engine cannot keep.

## `x3 scan`

```
x3 scan [-out <file>] [dir]
```

| Part | Meaning |
|---|---|
| `dir` | root directory to walk; defaults to `.` |
| `-out <file>` | write the JSON report to this file. Without it the report goes to **stdout** |
| (always) | human-readable findings and the summary line go to **stderr** |

Because the report goes to stdout and the findings to stderr, you can pipe the
JSON somewhere and still read the reds on your terminal.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | green — every directive found is well formed and in a legal scope |
| `1` | red — at least one directive failed; each one is printed with `file:line` |
| `2` | usage error, or the run could not complete (unparsable Go file, report not writable) |

A run over a tree that contains **no directives at all** exits `0`. That matters
if you build a gate on top of x3: the exit code alone cannot tell "everything
passed" from "nothing was checked". Read the counts in the report as well — see
[Using x3 from another project](#using-x3-from-another-project).

### What gets walked

- Only files ending in `.go` are read.
- These directory **names** are skipped: `vendor`, `testdata`, `node_modules`,
  and any directory whose name starts with `.` or `_`.
- The skip applies to sub-directories only. A skipped name given *as the root*
  is still scanned — that is how the control samples under
  `internal/scan/testdata/` get scanned:

```
x3 scan internal/scan/testdata/green   # exit 0
x3 scan internal/scan/testdata/red     # exit 1
```

### What a run looks like

Green sample, stderr:

```
x3 scan: 2 file(s) - 7 directive(s) - 0 red
```

Broken sample, stderr — one block per red, then the same summary line:

```
bad.go:3: unknown_category: no verifier exists for kind "nope"
	found: //x3:nope:whatever
bad.go:6: malformed: skip: expected shape //x3:skip:<reason>
	found: //x3:skip
bad.go:9: malformed: rule: expected shape //x3:rule:<kind>[:<subkind>...]
	found: //x3:rule:
bad.go:12: malformed: guard: expected shape //x3:guard:<kind>[:<subkind>...]
	found: //x3:guard
bad.go:15: malformed: allow: expected shape //x3:allow:<kind>:<reason>
	found: //x3:allow:secret
bad.go:18: malformed: case: the payload must start with in=(; expected shape //x3:case: in=(<args>) out=<want>
	found: //x3:case: in=1 out=2
bad.go:22: unattached: the directive binds to no declaration
	found: //x3:rule:idempotent
doc.go:1: scope_not_allowed: scope pkg is not allowed; valid scopes: decl
	found: //x3:case: in=(0) out=ErrInsufficientBalance
x3 scan: 2 file(s) - 8 directive(s) - 8 red
```

Paths are relative to the scan root and always use `/`, on every operating
system.

## Scopes

Where you write the directive decides what it binds. There are three legal
scopes and one failure state.

| Scope | Where you write it | Binds |
|---|---|---|
| `decl` | in the doc comment of a func, type, var, const or **import** | that one declaration; the report names it in `target` |
| `file` | above the `package` clause — the same placement as `//go:build` | that file |
| `pkg` | above the `package` clause **in a file named `doc.go`** | the whole package |
| `unattached` | anywhere else: inside a function body, or a floating comment | nothing — always red |

`pkg` is not a different syntax from `file`; it is the same placement in a file
named `doc.go`. That filename is the only thing that separates them.

Real resolutions from `internal/scan/testdata/green`:

```go
// doc.go:1 → scope "pkg"
//x3:guard:output:non-negative

// Package wallet, ...
package wallet
```

```go
// wallet.go:1 → scope "file"
//x3:live

package wallet
```

```go
// wallet.go:15 → scope "decl", target "Wallet.Add"
// Add, bakiyeye ekler ve yeni bakiyeyi döndürür.
//
//x3:rule:math:commutative
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

A method's `target` is written `Receiver.Method` with pointer stars and generic
brackets stripped: `*Wallet` and `Wallet[T]` both report `Wallet`.

And the failure state, from `internal/scan/testdata/red/bad.go:22` — a directive
inside a function body has nothing to attach to, so it is **not silently
ignored**, it is red:

```go
func Floating() {
	//x3:rule:idempotent
	_ = 1
}
```

## The dictionary

Six types are declared today. Every type states which scopes it is valid in and
what shape it must have. **A type that is not in the dictionary has no verifier,
and a directive with no verifier turns the run red** — invented types cannot
survive a scan.

| Directive | Valid scopes | Requires |
|---|---|---|
| `//x3:rule:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:guard:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:case: <payload>` | `decl` only | a non-empty payload |
| `//x3:live` | `decl`, `file`, `pkg` | nothing |
| `//x3:skip:<reason>` | `decl`, `file`, `pkg` | a reason |
| `//x3:allow:<type>:<reason>` | `decl`, `file`, `pkg` | a type **and** a reason |

**How a line is parsed.** After the `//x3:` prefix, the rest is split on `:`
into a category and its sub-types. A **payload** is whatever follows a colon
that is itself followed by a space — `: ` — and it runs to the end of the line:

| Written | Parsed as |
|---|---|
| `//x3:rule:math:commutative` | category `rule`, segments `["math","commutative"]` |
| `//x3:case: in=(0) out=Err` | category `case`, payload `in=(0) out=Err` |
| `//x3:skip: legacy generator` | category `skip`, payload `legacy generator` |

The payload counts toward the required-segment count, so a reason may be written
either way: `//x3:skip:legacy-generator` and `//x3:skip: legacy generator` are
both accepted. That is what lets a reason contain spaces.

---

### `//x3:rule:<type>[:<subtype>...]`

**Catches:** a semantic contract — a behavioural rule the code is expected to
obey. Today x3 checks only that you named a rule and named it in a legal scope;
nothing verifies that the rule actually holds.

**Green** — `internal/scan/testdata/green/wallet.go:15`:

```go
//x3:rule:math:commutative
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

**Red** — `internal/scan/testdata/red/bad.go:9`, a category with no sub-type:

```go
//x3:rule:
func EmptySegment() {}
```

```
bad.go:9: malformed: rule: expected shape //x3:rule:<kind>[:<subkind>...]
```

A trailing `:` is trimmed before parsing, so `//x3:rule:` is read as the bare
category `rule` — which needs one sub-type and does not have one. A doubled
colon such as `//x3:rule::idempotent` is red too, with
`empty subkind (a doubled colon)`.

---

### `//x3:guard:<type>[:<subtype>...]`

**Catches:** an invariant — a never-condition. Same shape and same scopes as
`rule`; the difference is meaning, not mechanics.

**Green** — `internal/scan/testdata/green/wallet.go:24`, bound to one method:

```go
//x3:guard:output:non-negative
func (w *Wallet) Withdraw(n int64) (int64, error) {
```

**Green** — `internal/scan/testdata/green/doc.go:1`, the same guard raised to
the whole package:

```go
//x3:guard:output:non-negative

// Package wallet, ...
package wallet
```

**Red:** there is no `guard` sample in `testdata/red` today. Its shape check is
the same code path as `rule`'s, so a bare `//x3:guard` fails exactly the way
`//x3:rule:` does above. That is a gap in the control samples, not a claim that
`guard` cannot go red — see [Gaps we know about](#gaps-we-know-about).

---

### `//x3:case: <payload>`

**Catches:** an inline example — one input and its expected output — written
next to the function instead of in a test file. **`decl` scope only**: an
example belongs to one declaration, so writing it at file or package level is
meaningless and therefore red.

The payload has a shape: `in=(<args>) out=<want>`. The argument list may be
empty — a call with no arguments is an example too — and the closing `)` is the
**last** one on the line, so a nested call fits: `in=(f(1), 2) out=ErrX`. A
payload that does not parse is `malformed`. What the parts *mean* is still not
checked: nothing calls the function and compares the result. That is the next
stage.

**Green** — `internal/scan/testdata/green/wallet.go:16`:

```go
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

**Red** — `internal/scan/testdata/red/doc.go:1`, a perfectly well-formed case in
the wrong scope:

```go
//x3:case: in=(0) out=ErrInsufficientBalance

// Package broken, ...
package broken
```

```
doc.go:1: scope_not_allowed: scope pkg is not allowed; valid scopes: decl
```

---

### `//x3:live`

**Catches:** code that talks to a real provider and costs money to exercise —
marked so it can be kept out of automated runs and used manually only.

**Green** — `internal/scan/testdata/green/wallet.go:1`, marking the whole file:

```go
//x3:live

package wallet
```

**Red:** `live` takes no sub-type and no reason, so it has no shape to get wrong
and cannot produce `malformed` on its own. Its failure modes are the two that
apply to every type: writing it where nothing can hold it (`unattached`) and a
doubled colon (`//x3:live::x`). No dedicated `live` red sample exists.

---

### `//x3:skip:<reason>`

**Catches:** a deliberate exemption. The **reason is mandatory** — a silent skip
is exactly the failure this engine exists to prevent, so a bare `//x3:skip` is
red rather than a free pass.

**Green** — `internal/scan/testdata/green/wallet.go:40`:

```go
//x3:skip: legacy generator
const legacyRate = 3
```

**Red** — `internal/scan/testdata/red/bad.go:6`, a skip with no reason:

```go
//x3:skip
func NoReason() {}
```

```
bad.go:6: malformed: skip: expected shape //x3:skip:<reason>
```

---

### `//x3:allow:<type>:<reason>`

**Catches:** a justified silence for one specific finding — the counterpart of
`skip` for scanners that flag things. It needs **two** parts: what is being
silenced, and why. One part alone is not enough.

**Green** — `internal/scan/testdata/green/wallet.go:35`, silencing a secret
finding on a constant that is deliberately not a real key:

```go
//x3:allow:secret:example-only
const demoToken = "not-a-real-key"
```

**Red:** no `allow` sample exists in `testdata/red` today. `//x3:allow:secret` —
a type with no reason — fails the same shape check as `//x3:skip` above.

## Error codes

Four codes, and they are the stable part of the output: the JSON `code` field is
what a machine should read, the `message` text may be reworded at any time.

| Code | Turns red when | Sample |
|---|---|---|
| `unknown_category` | the type is not in the dictionary — no verifier exists for it | `red/bad.go:3` — `//x3:nope:whatever` |
| `malformed` | a required sub-type or reason is missing, a doubled colon left an empty sub-type, or a `case` payload does not parse | `red/bad.go:6`, `red/bad.go:9`, `red/bad.go:18` |
| `scope_not_allowed` | the type is known and well formed, but may not be used in this scope | `red/doc.go:1` — a `case` at package level |
| `unattached` | the directive binds to nothing at all | `red/bad.go:22` — inside a function body |

The checks run in that order and stop at the first failure, so one directive
reports exactly one code. Everything that is not an error is counted `ok`.

## The JSON report

```json
{
  "version": 1,
  "root": "internal/scan/testdata/green",
  "files": 2,
  "directives": [ ... ],
  "summary": { "ok": 7, "errors": 0 }
}
```

`version` is the schema version — it goes up when the meaning of a field
changes. Directives are sorted by file, then by line, so two runs over the same
sources produce the same list in the same order.

One green entry and one red entry, verbatim:

```json
{
  "file": "wallet.go",
  "line": 1,
  "raw": "//x3:live",
  "category": "live",
  "scope": "file",
  "status": "ok"
}
```

```json
{
  "file": "bad.go",
  "line": 6,
  "raw": "//x3:skip",
  "category": "skip",
  "scope": "decl",
  "target": "NoReason",
  "status": "error",
  "code": "malformed",
  "message": "skip: expected shape //x3:skip:<reason>"
}
```

| Field | Notes |
|---|---|
| `file`, `line` | relative to the scan root, always `/`-separated |
| `raw` | the directive line exactly as written, trailing whitespace trimmed |
| `category`, `segments`, `payload` | the parsed line; `segments` and `payload` are omitted when empty |
| `scope`, `target` | resolved binding; `target` is present only for `decl` |
| `status` | `ok` or `error` |
| `code`, `message` | present only on `error` |

**There is no timestamp anywhere in the report, by design.** Identical sources
must produce identical bytes, so that a later ledger can compare two runs and
never raise a false red over a clock tick. `TestDeterministic` holds that line.

## `x3 lang`

A project that mixes languages outside its comments leaks the author's mother
tongue into identifiers, log lines and error messages, and nobody notices until
a stranger reads the code. `x3 lang` is the gate for that, and it is a **general**
capability: it knows nothing about which language you are leaking *from*.

**The dictionary runs in reverse.** There is no list of forbidden words — such a
list can only ever cover the language whose words somebody thought to write
down. What is known is the **allowed** language. Every token that is not in it is
red, whatever language it came from.

```
x3 lang [-config <file>] [-out <file>] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or I/O error. Without
`-out` the JSON report goes to stdout; findings always go to stderr.

### What is checked

| Read | Not read |
|---|---|
| the package name | comments (see `comments` below) |
| every **declared** identifier — function, type, variable, constant, struct field, parameter, result, label, import alias | the **use** of a name declared elsewhere |
| every string constant, struct tags included | import paths |

The asymmetry is deliberate. A name is spelled once, where it is declared, and
that is where the gate reads it; flagging every use would report the same word
fifty times. A name declared somewhere else — `fmt.Fprintf`, `pgx.Connect` — is
not yours to spell, so it is not yours to be red for.

### The token rule

A text is split into words on anything that is not a letter, and at case
boundaries, with runs of capitals kept together: `JSONPath` → `json` + `path`,
`wordCount` → `word` + `count`, `TOTAL` → `total`. A run of capitals stays whole
on purpose: split letter by letter, a foreign word written in capitals would
dissolve into fragments and slip through. Fragments shorter than three letters
are not read at all — `id`, `n`, `x3` are in no dictionary.

Each remaining word must be either in the embedded dictionary of the allowed
language or in `language.allow`. Then one absolute rule on top: **any non-ASCII
letter outside a comment is red**, whatever alphabet it belongs to, and
`language.allow` cannot excuse it. `ç`, `é`, `α`, `ш` are all evidence that a
second language got into the source.

### `language` in `x3.json`

```json
{
  "language": {
    "allowed": "en",
    "comments": "any",
    "allow": ["cfg", "ctx", "dsn", "json", "omitempty"]
  }
}
```

| Field | Meaning |
|---|---|
| `allowed` | the language of the source outside comments. `en` is the only embedded dictionary today; any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone — write them in your working language; `en` holds them to the same dictionary |
| `allow` | project terms and abbreviations that no dictionary has: `dsn`, `ctx`, `omitempty`, a product name. One ASCII word per entry, three letters or more — an entry that could never match a token is rejected rather than ignored |

**No file, or no `language` section, is not an error**: the smart default is
`allowed: "en"`, `comments: "any"`, no allow list. A `language` section that *is*
written and is wrong — unknown field, unknown language, dead allow entry — stops
the run. Fail-closed, like the `live` section.

### What a run looks like

`internal/lang/testdata/red/sample.go`, checked against a configuration with no
allow list:

```go
// Reason, hiçbir dilde kelime olmayan bir adı kullanır.
func Reason() string {
	notaword := "the reason is missing"
	return notaword
}

// Accented, ASCII dışı harf taşıyan bir dizgi sabiti.
const Accented = "café"
```

```
sample.go:8:2: not_in_dictionary: notaword (identifier)
sample.go:14:18: non_ascii_letter: é (string)
x3 lang: 1 file(s) - 2 finding(s) - dictionary "en"
```

The Turkish comments in that file are green: `comments` is `any`. The report
carries the same findings, sorted by file and line, with no timestamp:

```json
{
  "version": 1,
  "root": "internal/lang/testdata/red",
  "language": "en",
  "files": 1,
  "findings": [
    {
      "file": "sample.go",
      "line": 8,
      "column": 2,
      "where": "identifier",
      "token": "notaword",
      "code": "not_in_dictionary"
    }
  ]
}
```

`code` is the stable part — `not_in_dictionary` or `non_ascii_letter`; `where` is
`identifier`, `string` or `comment`.

### The embedded dictionary

141,848 words are compiled into the binary (`internal/lang/english.txt`, ~1.4 MB
of text). It is a custom list generated from the **English Speller Database**
(ESDB, formerly SCOWL) at <https://app.aspell.net/create>, size 70 (large), US
spelling, diacritics stripped, with the `hacker` special list included — which
is why `http`, `auth` and `err` are already words. The file was lowercased,
de-duplicated and cut to entries of three ASCII letters or more.

Its licence is permissive and requires the notice to travel with any copy, so
the notice is kept verbatim at the top of `english.txt` and repeated here:

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

### The control experiment

`check.ps1`, step `language gate`, runs the same binary four times and requires
all four answers:

| Run | Wants |
|---|---|
| the repository, with its own `x3.json` | `0` |
| `testdata/red`, no allow list | `1` — the planted word and the planted accent |
| `testdata/green`, with `testdata/allow.json` | `0` |
| `testdata/green`, with a configuration that has no `language` section | `1` — the allow list is what made it green |

The last row is the half that is easy to skip: an allow list that is never seen
to change an answer is decoration.

The repository holds itself to this gate. Its own `x3.json` lists 34 terms —
`cfg`, `ctx`, `dsn`, `fset`, `omitempty`, `pgx`, `testdata` and so on — which is
what the allow list is for. Writing that list is also how the gate paid for
itself the first time it ran: it found a misspelled field name in a test
fixture.

## `x3 arch`

A project's shape is written in prose — "the core does not know the modules",
"only the entry point wires them" — and prose does not fail a build. `x3 arch`
turns those sentences into rules the engine checks. The rules live in `x3.json`,
the verifier lives in the engine: neither do the rules enter the engine, nor do
the project's names enter a verifier.

**One rule kind is built:** `deps`, with two of its three matchers —
`match: "import"` reads the import graph, `match: "literal"` reads names the
compiler never sees. The other eight kinds and the `symbol` matcher are
specified in [ROADMAP-ARCH.md](ROADMAP-ARCH.md) and have no verifier yet. Naming one in `x3.json` stops the run with exit `2`
and says so — a planned kind that passed silently would be worse than no rule at
all.

```
x3 arch [-config <file>] [-out <file>] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or configuration error.
Without `-out` the JSON report goes to stdout; findings always go to stderr.

**No `arch` section means exit `2`**, deliberately the opposite of the language
gate's smart default. A language has a universal default; an architecture does
not, and an invented default architecture is the most dangerous silent green
there is.

### `arch` in `x3.json`

```json
{
  "arch": {
    "components": {
      "contract": ["internal/engine/**"],
      "shared":   ["internal/config/**", "internal/source/**"],
      "checkers": ["internal/arch/**", "internal/lang/**", "internal/live/**",
                   "internal/scan/**", "internal/testdb/**"],
      "entry":    ["cmd/**"]
    },
    "rules": [
      { "name": "the-contract-knows-no-implementation",
        "kind": "deps", "match": "import",
        "from": "contract", "deny": ["shared", "checkers", "entry"] },

      { "name": "only-the-entry-point-reaches-a-checker",
        "kind": "deps", "match": "import",
        "to": "checkers", "allowFrom": ["entry"] }
    ]
  }
}
```

That is this repository's own section, quoted from its `x3.json`. The engine
holds itself to it on every `check.ps1` run.

### Components

A component is a name and a set of path patterns. Membership is declared here,
by path — not labelled in the source — so the shape of the project is reviewed
in one place instead of being scattered over three hundred comments.

| Pattern | Matches |
|---|---|
| `**` | zero or more path elements; at the end of a pattern, **at least one** |
| `*` | a run inside one element, never crossing `/` |
| `?` | one character inside one element |

`internal/core/**` is what is *under* `internal/core`, not the directory itself;
`**/*.go` still sees a file at the root. What a single `*` catches is the
component's **instance** — `internal/modules/*/**` tells `alpha` from `beta` —
and that is what `except: "self"` compares.

Two boundaries the engine enforces on its own:

- **Paths resolve against the module root**, the directory holding `go.mod`, not
  against the directory you point the command at. Checking a subtree therefore
  does not invalidate the patterns. Without a `go.mod` above the root there is
  no way to turn an import path back into a directory, and the run stops.
- **A file has one component.** If two components' patterns claim the same file,
  or the same package, the run stops with exit `2`. A package that is silently
  counted on the wrong side is worse than a package with no component.

### The two rule forms

Every rule asks one of two questions, and mixing them is refused.

| Form | Written | Asks |
|---|---|---|
| outward | `from` + `deny` | may **this** component touch those? |
| inward | `to` + `allowFrom` | who may touch **this** component? |

```json
{ "name": "modules-must-not-know-each-other",
  "kind": "deps", "match": "import",
  "from": "modules", "deny": ["modules"], "except": "self" }
```

`except: "self"` narrows a component's ban to *other instances of itself*: a
module may reach into its own parts, not into its sibling's. It needs a single
star in that component's patterns to tell the instances apart, and it needs the
component to be in its own `deny` list; without either it would exempt
everything, so it is refused at load time.

In the inward form, the component's **own** files are always inside: an import
within a component is not access from outside. A file in no declared component
is outside, and named that way in the message.

### The `literal` matcher — names the compiler never sees

An import ban cannot catch a table, a queue or a bucket. The name is a string,
the code compiles, and the day its owner is removed the failure arrives at run
time. `match: "literal"` reads **string constants** in Go and **whole lines** in
every other configured file, so a name spelled in SQL, JSON or JavaScript is as
visible as one spelled in Go.

Two forms, and writing both is refused:

```json
{ "name": "resource-names-belong-to-their-owner",
  "kind": "deps", "match": "literal",
  "sources": ["**/*.go", "**/migrations/*.sql"],
  "pattern": "db_(?P<owner>[a-z0-9]+)_[a-z0-9_]+",
  "owner": "modules/${owner}",
  "alsoAllow": ["entry"] }
```

**Ownership** — `pattern` + `owner`. The owner is derived from the name itself:
the capture group named in `owner` says which instance of the component the name
belongs to, and only that instance (plus anything in `alsoAllow`) may spell it.
There is no hand-kept ownership list, because a list goes stale and then the
gate lies. `owner` may also be a bare component name, `"core"`, when the
component has no instances.

```json
{ "name": "the-core-names-no-queue",
  "kind": "deps", "match": "literal",
  "from": "core", "pattern": "queue_[a-z0-9_]+" }
```

**Prohibition** — `from` + `pattern`. This component may not spell a name
matching the pattern, whoever owns it.

| Field | Form | Meaning |
|---|---|---|
| `pattern` | both | a Go regular expression, compiled when the configuration loads |
| `owner` | ownership | `<component>` or `<component>/${<capture>}` |
| `alsoAllow` | ownership | components that may spell any owner's name |
| `from` | prohibition | the component that may not spell it |

Two boundaries:

- **Comments are not read.** Explaining why a rule exists requires naming the
  thing; what is forbidden is the *code* knowing it. In Go that is exact — the
  matcher reads string literals from the AST, not the file's text. In other
  files there is no syntax to lean on, so every line is read.
- **Exemptions cover Go only.** `//x3:allow:arch:` binds a declaration, an
  import or a file, and a `.sql` file has nowhere to write one. A finding in a
  text file is answered by fixing it, by naming the component in `alsoAllow`, or
  by narrowing `sources`.

An exemption above a declaration now covers **every line of that declaration**,
not just its first: a literal violation sits inside a function body, and an
exemption that only covered the signature would silence nothing.

### Fields a rule has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique in the file; what the report and the stderr lines call this rule |
| `kind` | yes | `deps` today; the other eight stop the run |
| `match` | yes | `import` or `literal`; `symbol` stops the run |
| `from` + `deny` | import | the outward question |
| `to` + `allowFrom` | import | the inward question |
| `pattern` + `owner` | literal | only the owner may spell this name |
| `pattern` + `from` | literal | this component may not spell it |
| `except` | no | `self` only, next to `from` + `deny` |
| `policy` | no | `warn` or `block`; **defaults to `block`**, the same law as live guards |
| `sources` | no | the file set this rule reads; defaults to `arch.sources`, and that to `["**/*.go"]`. The `import` matcher reads Go only; `literal` reads whatever the globs name |

Configuration is validated **strictly and up front**, as `live` already is: an
unknown key, a key belonging to another kind, a missing required key, a
duplicate `name`, an undeclared component name, an unknown `policy` or an empty
`rules` list stops the run with exit `2` before any rule executes. An empty list
is an error on purpose — a check with nothing in it is a silent pass.

### Exemptions

`arch` adds no directive type. A violation is silenced with the one the
dictionary already has:

```go
import (
	//x3:allow:arch: the ledger is wired to alpha here, and only here
	"example.com/app/modules/alpha"
)
```

- **`skip` does not silence `arch`.** Say what you are silencing by name, or a
  broad `//x3:skip:` would one day switch off the architecture too.
- **An exemption binds a line, not a tree.** It is written above a single import
  (it covers that import) or above the `package` clause (it covers the file).
  Above a parenthesised `import (` block it binds nothing and shows up dead: a
  block-wide silence is a deleted rule.
- **A reason is required.** `//x3:allow:arch` with nothing after it is malformed
  for `x3 scan` and silences nothing here.
- **Exemptions are listed in the report**, separately from violations. A silence
  nobody can see is not a silence, it is a loss.
- **A dead exemption is red.** An `allow:arch` that no violation needed reports
  `dead_exemption`. A stale exemption is how a gate goes quietly blind.

### Scope integrity

A rule that matched nothing is red — `empty_scope` — and this is engine
behaviour, not a rule you can choose to write. A gate holding a path constant
says "clean" and exits `0` the day the file it guards moves; it never saw it.

Every component the rule names is measured, the object side included: a `deny`
list pointing at a component with no files can never turn red, and that is a
silent pass wearing a green shirt. `empty_scope` and `dead_exemption` are always
`block`, whatever the rule's `policy` says — a policy grades how bad a violation
is, and neither of these is a violation. They are the measurement failing.

### What a run looks like

The planted violation in `internal/arch/testdata/red`, where `beta` reaches into
`alpha`:

```
BLOCK internal/arch/testdata/red/modules/beta/beta.go:3 (modules-must-not-know-each-other): forbidden_dependency
	component "modules" must not import another instance of itself
	internal/arch/testdata/red/modules/beta -> internal/arch/testdata/red/modules/alpha
x3 arch: 5 file(s) - 3 rule(s) - 1 block, 0 warn, 0 exempted
```

The same tree with the rule set to `policy: "warn"` prints `WARN` and exits `0`.
The same tree with the exemption in place prints the silence and exits `0`:

```
ALLOW core-must-not-know-modules internal/arch/testdata/exempt/core/ledger.go:4: the ledger is wired to alpha here, and only here
x3 arch: 2 file(s) - 1 rule(s) - 0 block, 0 warn, 1 exempted
```

### The arch report

No timestamp, and violations sorted by rule, then file, then line: the same
source produces the same bytes, so a later ledger never raises a red over a
clock tick.

```json
{
  "version": 1,
  "root": "internal/arch/testdata/red",
  "config": "internal/arch/testdata/arch-red.json",
  "files": 5,
  "rules": [
    { "name": "modules-must-not-know-each-other", "kind": "deps", "match": "import",
      "policy": "block", "subjects": 4, "violations": 1 }
  ],
  "violations": [
    { "rule": "modules-must-not-know-each-other", "kind": "deps",
      "file": "internal/arch/testdata/red/modules/beta/beta.go", "line": 3,
      "subject": "internal/arch/testdata/red/modules/beta",
      "object": "internal/arch/testdata/red/modules/alpha",
      "code": "forbidden_dependency", "policy": "block",
      "message": "component \"modules\" must not import another instance of itself" }
  ],
  "exemptions": [],
  "summary": { "rules": 3, "violations": 1, "warned": 0, "exempted": 0 }
}
```

`subject` and `object` are the packages; the message names the components.
`summary.violations` counts only `block` findings — those are what turn the run
red — while `warned` counts the rest.

### Error codes

The `code` field is the stable part; the `message` text may be reworded.

| Code | Raised by | Meaning |
|---|---|---|
| `forbidden_dependency` | `deps` | a forbidden import edge, or a component spelling a name it may not |
| `foreign_resource` | `deps:literal` | a component spelled a name another component owns |
| `empty_scope` | every rule | a component the rule names matched no file it reads |
| `dead_exemption` | exemptions | an `allow:arch` that no violation needed, or one that binds to no import |

The remaining nine codes in [ROADMAP-ARCH.md](ROADMAP-ARCH.md) belong to rule
kinds that do not exist yet.

### The control experiment

`check.ps1`, step `arch control experiment`, runs the same binary three times:

| Run | Wants |
|---|---|
| `testdata/green` with `arch-green.json` | `0` |
| `testdata/red` with `arch-red.json` | `1` — one planted violation, one finding |
| `testdata/literal-green` with its configuration | `0` — the owner spelling its own name is not a violation |
| `testdata/literal-red` with its configuration | `1` — three findings, one of them from a `.sql` file |
| this repository with its own `x3.json` | `0` |
| the same three rules split across three files | `1` — the parts carry the rules |

The Go tests carry the rest of the table: `warn` counts but does not stop the
run, an exemption silences and is listed, a dead exemption is red, an empty
component is red, two runs produce identical bytes, and fifteen broken
configurations are all refused before a rule executes.

A hand-written gate elsewhere is retired only after the `arch` rule has been
seen to go red on the **same** injected violation. Retiring without that double
red is forbidden.

## `x3 guard`

```
x3 guard [-config <file>] [-report <file>] [-stamp] -- <command> [args...]
```

Runs the **live guards** declared in a configuration file, then decides whether
the command after `--` may start. This is the *guard-then-launch* shape: one
process, one decision, no wrapper script.

| Part | Meaning |
|---|---|
| `-config <file>` | configuration file holding the guards; defaults to `x3.json` |
| `-report <file>` | write the JSON report here. **Without it no report is written** — stdout belongs to the launched command |
| `-stamp` | put a wall-clock start time in the report (off by default; see [The guard report](#the-guard-report)) |
| `--` | everything after it is the command and its arguments |
| (always) | the reason for every red guard, and the decision, go to **stderr** |

The command is started with x3's **own environment and working directory** —
nothing added, removed or rewritten — and its exit code is returned verbatim.
Its stdin, stdout and stderr are x3's own, so a launched test run or server
behaves exactly as it would without the guard in front of it.

### The decision rule

| Guards | Decision | What happens |
|---|---|---|
| all pass | `launch` | the command runs; x3 exits with the command's exit code |
| red, all of them `policy: warn` | `launch` | the command runs; each red is printed as `WARN` first |
| at least one red with `policy: block` | `blocked` | **the command is never started**; reasons go to stderr, x3 exits `1` |

A guard that could not run at all — missing environment variable, unreachable
host, unknown driver — counts as red. That is deliberate: a live guard whose
answer is unknown is not an answer, and the switch is **fail-closed**.

### Exit codes

| Code | Meaning |
|---|---|
| the command's own | the guards allowed the launch |
| `1` | a `block` guard was red, and the command was never started |
| `2` | the configuration could not be read or validated, the report could not be written, or the command could not be started at all |

`1` therefore carries two meanings — "blocked" and "the command itself exited
1". The report separates them: `decision` is `blocked` in the first case, and
`launch` with an `exit` field in the second. A gate that needs the distinction
passes `-report` and reads it.

## Live guards in `x3.json`

Guards are **declared, not coded**. There is no Go file per guard and no plugin
to write: the engine knows three general source kinds — `sql`, `http`, `exec` —
and everything project-specific (the query, the address, the expected value, the
policy) is data in the configuration file.

```json
{
  "$schema": "https://x3.example/x3.schema.json",
  "live": {
    "guards": [
      { "name": "...", "kind": "sql", "policy": "block", "...": "..." }
    ]
  }
}
```

Keys outside `live` are left untouched — `x3.json` is one file with several
sections: `language` (above) is read by `x3 lang`, and `settings`/`policies` are
waiting for later stages. Inside `live` the check is
**strict and up-front**: an unknown key, a key that belongs to a different kind,
a missing required key, a duplicate name, an unknown policy or an empty guard
list stops the run *before any guard is executed*. An empty list is an error on
purpose — a guard run with nothing in it would otherwise be a silent pass.

**Two surfaces, one law.** `//x3:live` written in source code is a *marker*:
this code talks to a real provider, keep it out of automated runs. The `live`
section here is where runnable guards are *defined*. Both are declared in a
dictionary inside the engine, and in both an entry the dictionary does not know
turns the run red. Invented kinds cannot survive, exactly as invented directive
types cannot.

### Fields every guard has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file; what the report and the stderr lines call this guard |
| `kind` | yes | `sql`, `http` or `exec` |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `timeoutMs` | no | time limit for this guard; defaults to `10000`. A dead dependency must not hang the gate forever |

### `kind: "sql"`

| Field | Required | Meaning |
|---|---|---|
| `dsnEnv` | yes | **name** of the environment variable holding the DSN. The DSN itself never appears in the file |
| `query` | yes | the query; its first row, first column is the observed value |
| `driver` | no | `database/sql` driver name; defaults to `pgx` |
| `equals` / `contains` | one of them | what the observed value must be |

`equals` or `contains` is mandatory here: a query with no expectation asserts
nothing, so the configuration is rejected rather than quietly passing.

```json
{
  "name": "schema-current",
  "kind": "sql",
  "policy": "block",
  "dsnEnv": "APP_DATABASE_URL",
  "query": "select max(version)::text from schema_migrations",
  "equals": "0117"
}
```

### `kind: "http"`

| Field | Required | Meaning |
|---|---|---|
| `url` | yes | the address; the request is always a `GET` |
| `status` | yes | the expected status code |
| `headerEnv` | no | header name → **name** of the environment variable holding its value |
| `jsonPath` | no | [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901) JSON Pointer into the response body, e.g. `/agent/permissions/0`. Without it the observed value is the whole body |
| `equals` / `contains` | no | what the observed value must be. With only `status`, the status code alone is the assertion |

```json
{
  "name": "provider-agent-enabled",
  "kind": "http",
  "policy": "warn",
  "url": "https://api.provider.example/v1/agents/self",
  "status": 200,
  "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
  "jsonPath": "/agent/permissions/0",
  "equals": "outbound"
}
```

### `kind: "exec"`

| Field | Required | Meaning |
|---|---|---|
| `command` | yes | executable to run |
| `args` | no | its arguments |
| `equals` / `contains` | no | what its trimmed stdout must be. Without either, **exit code 0** is the assertion |

```json
{
  "name": "toolchain-present",
  "kind": "exec",
  "command": "go",
  "args": ["env", "GOOS"]
}
```

### Secrets never enter the report

A live guard needs credentials, and a report is a file people paste into
tickets. So:

- Credentials are referenced **by environment variable name only** (`dsnEnv`,
  `headerEnv`). A DSN or a token written literally in `x3.json` is the caller's
  own mistake — x3 never asks for one.
- Before anything is written, the values of those variables are stripped out of
  the observed value **and out of the error text**. Driver errors routinely
  quote the DSN they failed on; that string is replaced with `[redacted]`.
- An empty environment variable is an error, not an empty credential: the guard
  goes red instead of asking anonymously and reporting a misleading `401`.

`TestSecretNeverLeaves` holds this line — it makes the fake driver fail with the
DSN inside its own error message and then asserts the password is nowhere in the
marshalled result.

## The guard report

```json
{
  "version": 1,
  "config": "internal/live/testdata/guard-block-red.json",
  "guards": [
    {
      "name": "toolchain-present",
      "kind": "exec",
      "policy": "block",
      "status": "pass",
      "expected": "exit code 0",
      "observed": "windows"
    },
    {
      "name": "impossible-platform",
      "kind": "exec",
      "policy": "block",
      "status": "fail",
      "expected": "equals \"there-is-no-such-platform\"",
      "observed": "windows",
      "detail": "observed value is not equal to the expected value"
    }
  ],
  "summary": { "pass": 1, "warned": 0, "blocked": 1 },
  "decision": "blocked",
  "command": ["x3", "scan", "internal"]
}
```

| Field | Notes |
|---|---|
| `version` | schema version of this report; it goes up when a field's meaning changes |
| `guards[].status` | `pass`, `fail` (it ran and disagreed) or `error` (it could not run). Both non-`pass` values are red |
| `guards[].policy` | the policy that was applied to **this** guard — always present, so the report explains its own decision |
| `guards[].expected` / `observed` / `detail` | what was wanted, what was seen, and why it counted as red. Secrets are already redacted |
| `summary` | `pass` + `warned` (red under `warn`) + `blocked` (red under `block`) |
| `decision` | `launch` or `blocked` |
| `command` | the command as given after `--`; absent when none was given |
| `exit` | the command's exit code. **Absent when `decision` is `blocked`** — that absence is the proof the command never ran |
| `startedAt` | present **only** with `-stamp` |

**No timestamp unless you ask for one**, the same rule as the scan report: the
same configuration and the same answers must produce the same bytes, so that a
later ledger comparing two runs never raises a red over a clock tick.
`TestReportIsDeterministic` holds that line.

### The control samples

Three configurations in `internal/live/testdata/` differ **only** in the guard
declaration; the launched command is identical in all three, and it proves it
ran by writing a file:

| Sample | Guards | Expected |
|---|---|---|
| `guard-green.json` | one passing `exec` guard | exit `0`, file written |
| `guard-block-red.json` | the same guard plus a red one, `policy: block` | exit `1`, **file not written** |
| `guard-warn-red.json` | the same pair, the red one `policy: warn` | exit `0`, file written, `WARN` on stderr |

`check.ps1` runs all three as the `guard control experiment` step and asserts
both the exit code and the presence of the file:

```
== guard control experiment
  green:     exit=0 ran=True (want 0/True)
  block-red: exit=1 ran=False (want 1/False)
  warn-red:  exit=0 ran=True (want 0/True)
```

And the stderr of the blocked run — one block per red guard, then the decision:

```
BLOCK impossible-platform (exec): observed value is not equal to the expected value
	want: equals "there-is-no-such-platform"
	got:  windows
x3 guard: 2 guard(s) - 1 pass, 0 warn, 1 block - command not started
```

The `sql` and `http` kinds are control-tested in `internal/live/live_test.go`
rather than in `check.ps1`, because the gate must not need a database or a
network: `TestSQLGuard` runs the real `database/sql` path against a fake driver
registered by the test, and `TestHTTPGuard` runs the real HTTP path against an
`httptest` server — each green, then each turned red by changing only the
expectation.

## `x3 guard:effective`

```
x3 guard:effective [-config <file>] [-out <file>] [-stamp]
```

A setting has two lives. One is the **record**: a row in a table, a field in a
remote configuration endpoint — the value somebody wrote down. The other is what
is **in force**: the value the running process actually loaded, and the value the
provider it talks to actually applies. The two drift apart quietly — a process
that was never reloaded, a remote setting edited by hand, a half-applied
migration — and nothing in the system complains, because every side is
internally consistent. This command reads the same setting from all of those
places at once and turns the disagreement red.

| Part | Meaning |
|---|---|
| `-config <file>` | configuration file holding the checks; defaults to `x3.json` |
| `-out <file>` | write the JSON report here; **stdout when empty** |
| `-stamp` | put a wall-clock start time in the report (off by default) |
| (always) | every divergent check, with what each source said, goes to **stderr** |

Unlike `x3 guard` this command launches nothing, so stdout is free for the
report — the same arrangement as `x3 scan` and `x3 lang`.

### The decision rule

| Checks | Exit | What it means |
|---|---|---|
| every source agrees | `0` | the setting on paper is the setting in force |
| divergent, all `policy: warn` | `0` | printed as `WARN`, the run is not stopped |
| at least one divergent `policy: block` | `1` | the record and the world disagree |
| a source could not be read at all | as above | **red** — an unknown answer is not an answer |

A check that could not read one of its sources is `error`, not a silent pass: if
the process endpoint is down, nobody can say whether it is running the recorded
model. That is the same fail-closed switch the live guards use.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | every `block` check agreed (a `warn` divergence still exits `0`) |
| `1` | at least one `block` check found a divergence or could not read a source |
| `2` | the configuration could not be read or validated, or the report could not be written |

## Effective checks in `x3.json`

Like the live guards, checks are **declared, not coded**, and they are built out
of the same three general source kinds — `sql`, `http`, `exec`. What changes is
the *role* a source plays: one is the record, the rest are the world.

```json
{
  "effective": {
    "checks": [
      {
        "name": "assistant-model",
        "policy": "block",
        "attempts": 3,
        "retryDelayMs": 500,
        "recorded": {
          "label": "database",
          "kind": "sql",
          "dsnEnv": "APP_DSN",
          "query": "select model from settings where id = 1"
        },
        "effective": [
          {
            "label": "process",
            "kind": "http",
            "url": "${APP_BASE}/internal/settings",
            "status": 200,
            "jsonPath": "/model"
          },
          {
            "label": "provider",
            "kind": "http",
            "urlEnv": "PROVIDER_SETTINGS_URL",
            "status": 200,
            "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
            "jsonPath": "/model",
            "map": { "engine-2-2026-01-31": "engine-2" }
          }
        ]
      }
    ]
  }
}
```

The `effective` section is read only by this command; `live`, `language` and
`dist` are untouched next to it. Validation is **strict and up-front**, with the
same fail-closed rules the `live` section has: an unknown key, a key belonging to
another kind, an unknown kind, a duplicate name, an empty check list, a check
with no `recorded` side or an empty `effective` list all stop the run before a
single source is read.

### Fields a check has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `attempts` | no | how many times the comparison is retried while it disagrees; defaults to `1` |
| `retryDelayMs` | no | wait between attempts; defaults to `250`. Only used when `attempts` is more than one |
| `recorded` | yes | one reading: the setting as it was written down |
| `effective` | yes | one or more readings: the setting as it is in force. All of them must equal `recorded` |

**Retries exist because the world lags the record.** A process reloads its
configuration a moment after the row changes; a provider propagates a change
through a cache. One attempt is the default precisely so that a retry is a
deliberate statement about how long the lag may be — never a way to wait out a
red.

### Fields a reading has

A reading is a `sql`, `http` or `exec` source — every field documented under
[Live guards in `x3.json`](#live-guards-in-x3json) applies unchanged, including
`timeoutMs`, `${ENV}` placeholders in `url`, `headerEnv` and `jsonPath`. Two
fields are added, and two are **not allowed**:

| Field | Meaning |
|---|---|
| `label` | the name this source carries in the report; defaults to `recorded` and `effective[0]`, `effective[1]`, … |
| `map` | value mapping applied before the comparison. A value the map does not mention is compared as it came |
| ~~`equals`~~ / ~~`contains`~~ | **rejected here.** A reading has no expectation of its own; its expectation is the other readings |

`map` is what makes two spellings of the same setting comparable: a provider that
answers `engine-2-2026-01-31` and a database row that says `engine-2` are the
same setting, and writing that down once is honest. A value the map does not
cover is *not* an error — it goes into the comparison unchanged, so an
incomplete mapping produces an explainable red, never a false green. The report
keeps the raw value next to the mapped one, so nobody has to guess what the
source actually said.

## The effective report

```json
{
  "version": 1,
  "config": "internal/live/testdata/effective-block-red.json",
  "checks": [
    {
      "name": "platform",
      "policy": "block",
      "status": "fail",
      "attempts": 1,
      "recorded": { "label": "record", "kind": "exec", "value": "windows" },
      "effective": [
        { "label": "world", "kind": "exec", "value": "amd64" }
      ],
      "detail": "record says \"windows\", but world says \"amd64\""
    }
  ],
  "summary": { "pass": 0, "warned": 0, "blocked": 1 }
}
```

| Field | Notes |
|---|---|
| `checks[].status` | `pass`, `fail` (the sources disagreed) or `error` (a source could not be read). Both non-`pass` values are red |
| `checks[].attempts` | how many attempts the answer actually needed; a `2` here says the world was late, not wrong |
| `recorded` / `effective[]` | `label`, `kind`, the compared `value`, the `raw` value when a mapping changed it, and `error` when that source could not be read. Secrets are already redacted |
| `detail` | the difference, source by source: `record says "windows", but world says "amd64"` |
| `summary` | `pass` + `warned` (red under `warn`) + `blocked` (red under `block`); `blocked` above zero is exit `1` |
| `startedAt` | present **only** with `-stamp`, the same determinism rule as the guard report |

Secrets follow the same law as the guards: credentials and addresses are named
by environment variable only, and their values are stripped out of every
observed value and every error message before anything is written.
`TestEffectiveSecretNeverLeaves` holds that line for this report specifically —
it points a reading at an address that only exists in the environment, lets the
transport error quote it, and asserts the address is nowhere in the marshalled
result.

### The control samples

Three configurations in `internal/live/testdata/` differ **only** in the
effective side of one comparison — the recorded side and the policy are the
knobs, the sources are the same two commands:

| Sample | Comparison | Expected |
|---|---|---|
| `effective-green.json` | both sides read `go env GOOS` | exit `0` |
| `effective-block-red.json` | the world side reads `go env GOARCH`, `policy: block` | exit `1` |
| `effective-warn-red.json` | the same divergence under `policy: warn` | exit `0`, `WARN` on stderr |

`check.ps1` runs all three as the `effective control experiment` step:

```
== effective control experiment
  green:     exit=0 (want 0)
  block-red: exit=1 (want 1)
  warn-red:  exit=0 (want 0)
```

And the stderr of the blocked run names every source, not just the verdict:

```
BLOCK platform: record says "windows", but world says "amd64"
	record (exec): "windows"
	world (exec): "amd64"
x3 guard:effective: 1 check(s) - 0 pass, 0 warn, 1 block
```

**Mapping and retries are control-tested in `internal/live/effective_test.go`**,
because both need answers the gate cannot arrange with a shell command:

- `TestEffectiveMapping` — the same two sources, one answering `engine-2` and the
  other `engine-2-2026-01-31`. Without `map` the check is **red**; with `map` it
  is **green**. A mapping whose removal changes nothing is doing nothing.
- `TestEffectiveRetry` — a server that answers with the stale value once and the
  fresh value afterwards. `attempts: 1` is **red**, `attempts: 2` is **green**,
  and the report says it took two.
- `TestEffectiveUnreadableSourceIsRed` — one source pointed at an address nobody
  answers; the check is `error`, not a pass.

## `x3 version`

```
x3 version
```

Prints the release tag embedded in the binary at build time, and exits `0`:

```
v0.1.0
```

One line, nothing else, so a gate can compare it with the version it pinned
without parsing anything. A binary that was not produced by a release run has
no tag to embed and prints `unreleased`; an untagged binary is not a published
one, and a gate that pins versions should treat it as red.

## `x3 testdb`

```
x3 testdb create [-config <file>]
x3 testdb drop   [-config <file>] (-name <database> | -stale)
x3 testdb list   [-config <file>] [-stale]
x3 testdb run    [-config <file>] [-keep] -- <command> [args...]
```

Tests that share one database serialise on it, and tests that build their own
schema pay for the migrations every time. `x3 testdb` gives a run its own
PostgreSQL database, cheaply: it **clones a prepared template** instead of
replaying the migrations, hands the command a DSN through the environment, and
drops the database when the command is done.

| Subcommand | What it does |
|---|---|
| `create` | makes a database and prints its **DSN on stdout**, one line and nothing else, so a shell can capture it |
| `drop -name <db>` | drops one database. The name must be one x3 created — see below |
| `drop -stale` | drops every leftover older than `maxAgeMinutes` |
| `list` / `list -stale` | prints `name` and age, one per line, for the databases x3 created |
| `run -- <command>` | creates, runs, drops. The command's exit code is returned verbatim |

`run` is the shape most projects want: one process, a fresh database, automatic
cleanup even when the command fails. `-keep` leaves the database behind for
inspection, which is exactly what `drop -stale` later collects.

### Two invariants

**Speed.** A database per test package is only worth having if creating one costs
milliseconds, so the intended setup is a template database that already carries
the schema, cloned with `CREATE DATABASE … TEMPLATE …`. Measured on PostgreSQL
18.2 over a loopback connection, averaged over five runs of a 40-table,
40-index schema: **238–263 ms** to clone the template against **275–292 ms** to
create an empty database and replay the same DDL. End to end — process launch,
connection, `CREATE DATABASE` — `x3 testdb create` took **409 ms**. The gap
widens with the schema: cloning is one directory copy whatever the migration
count, while replaying grows with it.

**Safety.** Every name this command touches has to be one x3 made. Two rules,
both checked before a single byte reaches the server:

1. the name matches `^[a-z_][a-z0-9_]{0,62}$` — `CREATE DATABASE` takes no bound
   parameters, so the name is text inside a statement, and this pattern is the
   injection gate, not a style rule;
2. the name carries the configured `prefix` **and** the creation stamp x3 writes
   into it.

A name that fails either rule exits **1** — the gate refused it — while a name
that passes and then cannot be reached exits **2**. That difference is what makes
the gate observable from outside, and `check.ps1` measures exactly it.

### `testdb` in `x3.json`

```json
{
  "testdb": {
    "adminDsnEnv": "APP_ADMIN_DSN",
    "template": "app_test_template",
    "prefix": "apptest_",
    "dsnEnv": "APP_TEST_DSN",
    "maxAgeMinutes": 120,
    "migrate": { "command": "./migrate", "args": ["up"], "timeoutMs": 60000 }
  }
}
```

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the environment variable holding the maintenance DSN. Point it at a maintenance database (`postgres`), never at the template: a template with an open connection cannot be cloned |
| `driver` | no | `database/sql` driver name; defaults to `pgx` |
| `prefix` | no | name prefix, defaults to `x3test_`. It is also the **authority boundary**: nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | template database to clone. Without it an empty database is created and the migration hook does the work |
| `dsnEnv` | no | name of the variable the new DSN is exported as, for the hook and for `run`; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover counts as stale; defaults to `120` |
| `migrate` | no | `command`, `args`, `timeoutMs`. Run after creation with the DSN in the environment |

The DSN handed to the command is the maintenance DSN with **only the database
name changed**, so credentials and connection options carry over. Both PostgreSQL
spellings are understood — the URL form (`postgres://…`) and unquoted
`key=value` pairs.

**The creation time is in the name.** PostgreSQL does not record when a database
was created, so x3 encodes the moment into the name it generates
(`<prefix><base36 seconds>_<random>`). That is what lets `list -stale` and
`drop -stale` work on any server with no extra table and no privileges beyond
creating databases — and it is also why a database x3 did not name has no age
and is therefore never touched.

**If the migration hook fails, the database is dropped.** A half-built schema is
worse than none: the run would fail somewhere further along and blame the wrong
thing.

### Secrets and errors

The maintenance DSN is named by environment variable only, and its value —
together with the password inside it — is stripped out of every error message
before it is printed; drivers routinely quote the connection string they failed
on. `TestAdminSecretNeverLeaves` holds that line. The DSN of the *created*
database is deliberately printed by `create`, because that is the whole point of
the subcommand; under `run` it is never printed, only passed through the
environment. What the launched command itself prints is its own business.

### The control experiment

The gate must run without a database, so the sample that runs in `check.ps1` is
the safety gate, proven in both directions with the server deliberately
unreachable:

```
== testdb control experiment
  foreign name refused at the gate: exit=1 (want 1)
  own name reached the server:      exit=2 (want 2)
```

A gate that refused *every* name would also exit `1` on the first line; the
second line is what rules that out.

The rest is control-tested in `internal/testdb/testdb_test.go` against a fake
`database/sql` driver that records the statements it is given — because for
these invariants it is not enough that a call returned an error, it has to be
seen that **nothing reached the server**:

- `TestDropRefusesForeignNames` — our own generated name produces a
  `DROP DATABASE`; `postgres`, `production`, a name with a semicolon in it and
  four other shapes produce **no statement at all**.
- `TestCreateClonesTheTemplate` — with `template` configured the statement
  carries `TEMPLATE`, without it the statement does not.
- `TestListIgnoresForeignNames` — a catalogue holding two x3 names and two
  hand-made ones yields two databases, and the ages come out of the names.
- `TestMigrationFailureDropsTheDatabase` — a hook that exits non-zero leaves no
  database behind.
- `TestRunDropsAfterTheCommand` — the drop happens after the command, and
  `-keep` suppresses it.

## Splitting the configuration

One `x3.json` is enough for a small repository and wrong for a large one. A code
base with dozens of gates puts thousands of lines into one file, and the rules
that belong to a module end up far from the module — so removing the module
leaves its rules behind, guarding nothing.

The root file declares its parts:

```json
{
  "include": ["x3/*.json", "apps/*/x3.json"],
  "language": { "allowed": "en" }
}
```

Every command reads the merged result. There is one merge law, and it knows
nothing about any section's schema:

| Both sides are | Result |
|---|---|
| lists | the parts are **added**, root first, then the files in name order |
| objects | merged key by key, recursively |
| anything else | **refused** — the run stops and names both files and the key |

So `arch.rules` from four files become one list, `arch.components` from two
files become one object, and two files setting `language.allowed` stop the run.
Nothing is silently overwritten: a setting that quietly loses to another file is
a setting whose author believes it is in force.

Four more refusals, all of them fail-closed:

- **A pattern that matches no file.** An `include` that was written and does not
  work is a set of rules nobody notices is missing.
- **A part that includes.** Parts are one level deep, so the whole configuration
  is readable from the root file. Nesting hides where a rule came from.
- **Discovery.** Parts are declared, never found by scanning a directory: a file
  dropped into a folder must not add a rule nobody reviewed.
- **A missing section** is still an error for the command that needs it, exactly
  as with a single file.

Ordering is by file name, so the merged configuration is the same on every run
and on every machine.

A project that does not split pays nothing: without an `include` key the file is
read exactly as before.

## Pilot: a real `x3.json`

x3 is piloted inside a real production application. Nothing about that
application is encoded in the engine; what follows is its configuration file,
with generic names, as an example of what live guards are actually for.

The pilot's problem is the one every deployment has: a long test or migration
run that starts against a **wrong live environment** wastes an hour and can
corrupt state. Four questions must be answered before it starts.

```json
{
  "live": {
    "guards": [
      {
        "name": "schema-current",
        "kind": "sql",
        "policy": "block",
        "dsnEnv": "APP_DATABASE_URL",
        "query": "select max(version)::text from schema_migrations",
        "equals": "0117"
      },
      {
        "name": "catalog-engine-address",
        "kind": "sql",
        "policy": "block",
        "dsnEnv": "APP_DATABASE_URL",
        "query": "select engine_ref from capability_catalog where tier = 'standard'",
        "equals": "provider:engine-v3"
      },
      {
        "name": "provider-agent-permission",
        "kind": "http",
        "policy": "warn",
        "url": "https://api.provider.example/v1/agents/self",
        "status": 200,
        "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
        "jsonPath": "/agent/permissions/0",
        "equals": "outbound"
      }
    ]
  }
}
```

| Guard | The question it answers | Why that policy |
|---|---|---|
| `schema-current` | is the **migration ledger** at the schema version this code expects? | `block` — running against an older schema produces failures that look like code bugs and are not |
| `catalog-engine-address` | does the **catalog row for this tier** still point at the engine address the run assumes? | `block` — a stale row silently routes the whole run somewhere else |
| `provider-agent-permission` | does the **provider still grant this agent the permission** the run needs? | `warn` — an external provider having a bad minute should not stop local work, but nobody should discover it an hour in |

The gate then becomes one line, and there is no shell logic deciding anything:

```
x3 guard -config x3.json -report build/guards.json -- go test ./...
```

**The red that made this worth building.** When `PROVIDER_TOKEN` holds a rotated
key, the external endpoint answers `401`, and the run says so before anything
starts:

```
WARN  provider-agent-permission (http): want status 200, got 401
	want: status 200 and equals "outbound"
	got:  status 401
x3 guard: 3 guard(s) - 2 pass, 1 warn, 0 block - starting go
```

The token itself appears nowhere — not in the config, not on stderr, not in
`build/guards.json`. Change that guard's policy to `block` and the same
situation stops the run instead of warning about it; that one word is the whole
difference.

## Releases and reproducible builds

The engine is published as binaries — one per platform — into a public
repository that carries nothing else: the two binaries, `SHA256SUMS.txt` and a
generated `README.md`. The source repository is private, so the binary and its
documentation are the whole public surface.

One command produces a release, and if any step of it fails nothing is
published:

1. it builds `windows/amd64` and `linux/amd64` with
   `-trimpath -buildvcs=false -ldflags "-s -w -buildid= -X main.version=<tag>"`
   and `CGO_ENABLED=0`, so the binary carries no build path, no build id and no
   VCS stamp — only the tag;
2. it builds **each target a second time** and compares the SHA256 of the two
   passes. A build that does not reproduce is not published, and that
   comparison happens on every release rather than in a one-off experiment;
3. it writes `SHA256SUMS.txt` and generates the public `README.md` from this
   document plus a template, stamping the release tag and the SHA256 of both
   sources into the generated file;
4. it re-reads what it just wrote and runs the staleness gate against it.

**The staleness gate.** The gate recomputes the SHA256 of this document and of
the README template and compares them with the stamp in the published README.
Either one changing after the last release run turns the step **red**: the
binaries do one thing and the README describes another. A publish directory
that is not configured, or configured and missing, is red as well and says
`NOT GENERATED` — deliberately not a green skip, because "nobody has published
yet" and "the publication is current" are not the same answer. The step carries
its own control experiment: it asks the same question again with a deliberately
wrong document hash and requires a red answer.

The publish directory is configuration and never a constant in the code: the
`-DistDir` argument wins, then the `X3_DIST_DIR` environment variable, then
`dist.dir` in `x3.json`, resolved relative to the repository root.

**Checking a downloaded binary.** The published `SHA256SUMS.txt` is in the
format `sha256sum -c` reads. A consuming project pins the tag and the checksum,
not a path (see below).

## Using x3 from another project

x3 is being piloted inside a real production Go application. The engine stays
general: nothing about that application is encoded in the engine or in this
document. What follows is how the integration works in practice, and it is the
same for any project.

**Integration is by binary, not by import.** The consuming project does not add
x3 to its `go.mod`, does not use a `replace`, and does not put x3 in a
`go.work`. It builds the binary and calls it:

```
go build -o <path> ./cmd/x3      # in the x3 checkout
<path> scan <package-or-tree>    # from the consuming project's gate
```

The directives are plain comments, so the consuming project's compiler never
sees them and its dependency graph never learns that x3 exists.

**Pin a version and a checksum, not a path.** The consuming project records
the release tag and the SHA256 of the binary it verified against — one small
tracked file — and fetches that exact file from the public binary repository
into a directory its VCS ignores. A checked-in path (or an environment variable
holding one) is green on the machine that wrote it and unmeasured on every
other one, and neither of them can tell you *which* build ran.

**A missing or mismatched binary is red, not skipped.** The pilot's gate was
fail-open at first: no binary meant a warning and a normal start. That is the
failure this engine exists to prevent, so it now refuses — no binary, wrong
version, wrong checksum, all three stop the run and print the command that
fetches the pinned release. "The tool was not there" and "the tool found
nothing" must never produce the same colour.

**Read the counts, not only the exit code.** This was measured on the pilot: a
scan of a tree with no directives in it exits `0` and reports `0 red`. A
gate that trusts the exit code alone turns "delete the directives" into a way to
go green. The pilot's gate therefore asserts on the report itself — it requires
the expected directives to be present and verified, and goes red if the count
drops.

**Directives arrive next to the existing tests, not instead of them.** In the
pilot the existing test file was kept untouched and the directives were added
alongside it. Nothing is migrated until its x3 equivalent has been seen to go
red on a deliberately broken input.

## Gaps we know about

Stated plainly, because a capabilities document that lists only strengths is a
sales page.

- **The language gate speaks one language.** `en` is the only embedded
  dictionary, so `allowed` accepts nothing else today. A dictionary is also a
  blunt instrument: an English word the list does not have (a rare technical
  term) is red until it is allow-listed, and a foreign word that happens to be
  an English word (`kilim`, `sultan`) passes. The non-ASCII rule is what catches
  most of the second case.
- **`arch` reads dependencies and nothing else yet.** One of nine rule kinds is
  built. A capability written where it does not belong — cryptography, an
  outbound request, retry logic — needs `deps:symbol`, which is specified and
  does not exist. The `import` matcher still reads Go only, so pointing its
  `sources` at a text glob leaves it with nothing to read.
- **A text file cannot carry an exemption.** `//x3:allow:arch:` is a Go comment.
  A `literal` finding in SQL or JSON is answered by fixing it, by `alsoAllow`,
  or by narrowing `sources` — not by silencing that one line.
- **The inward form does not see a component's inside.** `to` + `allowFrom`
  answers "who reaches in from outside", so one checker importing another
  checker inside the same component passes. Splitting them into instances needs
  a single star in the pattern, which a list of fixed directories cannot have;
  until the pattern language grows, that rule is written as one `deny` rule per
  component or not at all.
- **`case` payloads are parsed but not run.** The `in=(...) out=...` shape is
  checked; the values in it are not. Nothing calls the function and compares the
  result, so a `case` that is well formed and wrong stays green.
- **Only one of the four components exists as behaviour.** Recorder and Ledger
  are interfaces in `internal/engine` with no implementation, so the "nothing
  unchanged is ever re-checked" property described in the README is not real
  yet. Live guards are outside that picture: they check the environment, and
  nothing about them is remembered between runs.
- **The gate's guard control experiment covers `exec` only.** `sql` and `http`
  are proven red and green in `internal/live/live_test.go`, against a fake
  driver and an `httptest` server. Neither has ever been seen red against real
  infrastructure inside `check.ps1`, because the gate must run without a
  database or a network.
- **Effective checks compare strings too**, through the same `equals` machinery
  in reverse: two sources agree when their mapped values are byte-identical. A
  value that differs only in case, in whitespace inside the text, or in numeric
  formatting (`1` against `1.0`) needs a `map` entry, and a setting that is a
  list or an object cannot be compared at all — only the single value a
  `jsonPath` or a query cell yields.
- **`map` is a lookup table, not a rule.** Every spelling a source may answer has
  to be written down; there is no pattern, prefix or version-range form, so a
  provider that appends a fresh date to its identifier needs a new entry each
  time. The failure mode is a red that names both spellings, which is the safe
  direction.
- **`x3 testdb` speaks PostgreSQL only.** `CREATE DATABASE … TEMPLATE`,
  `DROP DATABASE … WITH (FORCE)` (PostgreSQL 13 and later) and `pg_database` are
  written into the commands, so another engine needs another implementation, not
  another `driver` value.
- **Nothing prevents two runs from sharing a template.** The template is read
  concurrently, which PostgreSQL allows, but a template being *rebuilt* while a
  clone starts is a race x3 does not arbitrate.
- **`testdb` keeps no record of its own.** Its whole memory is the name it
  generates, so a run killed between `CREATE` and the drop leaves a database
  that only `drop -stale` will notice, and only after `maxAgeMinutes`.
- **Effective checks never remember.** Each run compares the present answers;
  nothing is stored, so "this drifted three hours ago" is not a question the
  command can answer.
- **Guard expectations are string comparisons.** `equals` and `contains`, and
  nothing else: no regular expressions, no numeric or version ordering, so
  "schema at least 0117" cannot be written today — only "schema is 0117".
- **`http` guards are `GET` only**, with no request body and no redirect or
  TLS policy of their own.
- **Guards run one after another**, in file order, each with its own timeout.
  Ten slow guards take the sum of their times.
- **The guard report is written once, after the command finishes.** If x3 is
  killed while the launched command is running, no report file is produced,
  even though the guards did run.

## The documentation gate

This file is enforced. `check.ps1` runs a `docs gate` step:

> If the diff under review touches anything under `internal/` or `cmd/`, it must
> also touch something under `docs/`. Otherwise the gate is **red**.

The diff under review is:

- the **uncommitted changes** — staged, unstaged and untracked — when the
  working tree is dirty. This is the pre-commit case.
- otherwise the **HEAD commit**. This is the after-commit and CI case.

**Justified exemption.** A code change that genuinely needs no documentation
update is committed with a reason in the commit body:

```
docs: none — <reason>
```

`docs: yok — <neden>` is accepted as well. The reason is required: the line must
carry actual words, not just the marker. The gate reads the body of the commit
under review, so an exemption is a permanent, reviewable part of the history
rather than a flag someone passed once.

For a pre-commit run, where no commit body exists yet, the same exemption can be
given for that one run through the `X3_DOCS_NONE` environment variable — its
value is the reason, and the gate prints it, so an exemption is never silent.

**The gate itself was control-tested**, as every verifier here must be: a
capability file was edited with no documentation change and the gate went
red; the edit was reverted and the gate went silent. A gate whose red has never
been seen is not a gate.

---

<!-- x3-dist version=v0.5.0 capabilities=aa2a1bc06e59264429f6c0b2171ad2957740cf90c3339c26babdd72e294b234e template=09bd5c3267b29248c7fcaf7ceb6f1350b1567210cb146095452e47a10d69d713 -->
