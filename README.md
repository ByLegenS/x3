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

**Current version: `v0.36.1`**

## Download

| File | Platform | Size | SHA256 |
|---|---|---|---|
| `x3-windows-amd64.exe` | windows/amd64 | 13.3 MB | `d0fc867169c83c58dc994040b6edfa8e3cfba62d806593fd91b9889ee115a773` |
| `x3-linux-amd64` | linux/amd64 | 12.9 MB | `38c6546ece84d1cbf07b1f9d6fc9e8c79cd3e60f091cbc3710d8e1e74a165acc` |

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
x3 docs  -config x3.json .      # changes that must not travel alone
x3 secrets -config x3.json .    # credentials that got into the source
x3 boxes -config x3.json .      # open work, measured against its criteria
x3 syntax -config x3.json .     # files no compiler reads, parsed anyway
x3 scope  -config x3.json .     # a change that must stay in its lane
x3 record -listen :9100 -target http://localhost:8080 -ledger api.jsonl  # traffic, written down
x3 replay -target http://localhost:8080 -ledger api.jsonl          # and compared with it
x3 outbound serve -listen :9101 -ledger out.jsonl                  # the far side, from the ledger
x3 guard -config x3.json -- go test ./...   # live checks, then the command
x3 guard:effective -config x3.json          # the setting on paper vs in force
x3 testdb run -config x3.json -- go test ./...   # a fresh database for this run
```

Exit codes are the same for every command: **0** green, **1** red, **2** usage
or I/O error. When `guard` launches the command, the command's own exit code is
returned instead.

Project configuration lives in one file, `x3.json`: the `language` section for
the language gate, the `arch` section for the architecture rules, the `freeze`
section for the frozen baselines, the `docs` section for coupled changes,
the `secrets` section for the leak scan,
the `boxes` section for the open-work list, the `record` section for
what a recording must hide, the `replay` section for what may differ,
the `cache` section for where a run may remember what it measured, the `live`
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
- [Expectations](#expectations) — the count a scan must reach, so deleting directives cannot go green
- [`x3 lang`](#x3-lang) — the language gate: one language outside comments, dictionary in reverse
- [`x3 arch`](#x3-arch) — architecture rules: which component may import which
- [`x3 freeze`](#x3-freeze) — frozen sets that are only allowed to shrink
- [The count mode](#the-count-mode) — a number per key, and the cap that takes no debt
- [The finding baseline](#the-finding-baseline) — today's findings frozen, tomorrow's red
- [`x3 docs`](#x3-docs) — changes that must not travel alone
- [`x3 secrets`](#x3-secrets) — credentials that got into the source
- [Excluding what the pattern also catches](#excluding-what-the-pattern-also-catches) — per-pattern exclusions, because RE2 has no lookaround
- [`x3 comments`](#x3-comments) - the comment diet: block limits, and a ratio that only warns
- [`x3 boxes`](#x3-boxes) — an open-work list the machine can read
- [`x3 syntax`](#x3-syntax) - files nobody compiles, parsed before they ship
- [`x3 scope`](#x3-scope) - a change that must stay in its lane
- [`x3 record`](#x3-record) — a run of the application written down, redacted before the disk
- [`x3 replay`](#x3-replay) — the recording, sent again and compared field by field
- [`x3 guard`](#x3-guard) — run live guards, then launch a command only if they pass
- [Choosing which guards run](#choosing-which-guards-run) — tags, `-only` and `-skip`: a subset out of the one file everybody reviews
- [`x3 version`](#x3-version) — the release tag embedded in the binary
- [`x3 update`](#x3-update) — the binary replaces itself from a release, checksum first
- [`update.pin`](#updatepin--the-checksum-the-project-itself-vouches-for) — the checksum kept in the project, because a release cannot vouch for itself
- [Live guards in `x3.json`](#live-guards-in-x3json) — the three source kinds and the warn/block switch
- [The guard report](#the-guard-report)
- [`x3 guard:effective`](#x3-guardeffective) — compare a setting as recorded with the same setting as it is actually in force
- [Effective checks in `x3.json`](#effective-checks-in-x3json) — the recorded side, the effective sides, mapping and retries
- [The effective report](#the-effective-report)
- [`x3 testdb`](#x3-testdb) — run-lifetime test databases: clone, migrate, drop, collect the leftovers
- [Speed](#speed) — every core, same bytes, and the content-keyed cache
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
| **Recorder** (`internal/record`) | **implemented** — a reverse proxy records inbound HTTP; the interface in `internal/engine` still describes the wider goal |
| **Ledger** (`internal/record`) | **implemented** — JSON Lines, one interaction per line, redacted before it is written |

Alongside them, one capability that is not part of that four-component picture:

| Capability | State |
|---|---|
| **Live guards** (`internal/live`) | **implemented** — `sql`, `http` and `exec` checks declared in `x3.json`, a `warn`/`block` policy each, and the `x3 guard` command that launches a command only when they allow it |
| **Language gate** (`internal/lang`) | **implemented** — `x3 lang` checks that everything outside comments is written in one language, against an embedded English dictionary plus the project's own `language.allow` list |
| **Effective checks** (`internal/live`) | **implemented** — `x3 guard:effective` reads one setting from the place it is *recorded* and from every place it is *in force*, and turns a divergence red |
| **Architecture rules** (`internal/arch`) | **implemented** — `x3 arch` compares the import graph against the components and rules a project declares in `x3.json`; all nine specified rule kinds have a verifier |
| **Frozen baselines** (`internal/freeze`) | **implemented** — `x3 freeze` measures a set, compares it with a baseline the repository keeps, and turns growth red; `-update` records a shrink and refuses to record growth |
| **Coupled changes** (`internal/docs`) | **implemented** — `x3 docs` reads what a diff touched and asks for the counterpart change the project declared; the exemption needs a written reason |
| **Secret scan** (`internal/secrets`) | **implemented** — `x3 secrets` searches every text file for credential formats, masks what it finds, and takes a reasoned `//x3:allow:secret:` as the only silence |
| **Open work** (`internal/boxes`) | **implemented** — `x3 boxes` measures each box in the project's work list against the criteria that would prove it done, and reds both a finished box left open and a closed box with nothing to show |
| **Comment diet** (`internal/comments`) | **implemented** - `x3 comments` measures comment blocks against a limit and turns a long one red; the ratio of comment to code only warns, because the measure is necessity rather than count |
| **Syntax** (`internal/syntax`) | **implemented** - `x3 syntax` parses the files no compiler reads (a built-in JSON parser, or a parser the project names), and refuses to go green when the parser it was told to use is not installed |
| **Lane discipline** (`internal/scope`) | **implemented** - `x3 scope` reads what a change touched and turns a commit red when it enters a declared lane and also reaches outside it; crossing needs a reason in the message |
| **Recorded traffic** (`internal/record`) | **implemented** — `x3 record` stands in front of the running application, passes the traffic through untouched and writes it down with credentials, matched secret patterns and declared fields already masked; `x3 replay` sends the recording again and compares status, declared headers and body field by field |
| **Incremental cache** (`internal/cache`) | **implemented** — a run remembers what it measured, keyed on engine version, configuration fingerprint and file content; declared per project, off when it is not declared |
| **Test databases** (`internal/testdb`) | **implemented** — `x3 testdb` clones a template database per run, applies a migration hook, drops it when the command finishes, and collects what earlier runs left behind |

What is implemented is a **language check**, not a behaviour check. The scanner
answers three questions about every `//x3:` line it finds:

1. Is this directive type known — does it have a verifier at all?
2. Is its shape right — are the required sub-types and the reason present?
3. Is it in a scope where this type is legal?

It never calls your code, never runs a case, never proves that a `rule` holds.
(`x3 record` and `x3 replay` do reach behaviour — a recorded run compared with a
later one — but only through the HTTP surface, and only over the traffic the
recording happened to see.)
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

## Expectations

A scan reports what it finds. It cannot report what should have been there and
was not, and that gap has a name: **deleting the directives is a way to go
green.** A tree with no directives in it scans clean, reports `0 red` and exits
`0` - correctly, because nothing in it is wrong. Nothing in it is checked
either, and the exit code cannot tell those two apart.

An expectation closes that. The project declares how many verified directives a
scan must find, and `x3 scan` counts its own report:

```json
{
  "expect": [
    {
      "name": "the ledger package keeps its guards",
      "paths": ["ledger/**"],
      "category": "guard",
      "kind": "lookup",
      "min": 2
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `name` | required; the red names the expectation that was not met |
| `min` | required, at least 1 - an expectation of zero verifies nothing |
| `paths` | glob patterns, **relative to the scan root**; absent means the whole scan |
| `category` | `guard`, `rule`, `case`, `live`, `allow`, `skip`, ...; absent means any |
| `kind` | the first segment after the category (`guard:lookup` -> `lookup`); absent means any |

An unmet expectation is red and says what it counted:

```
BLOCK expectation_not_met: the ledger package keeps its guards
	the ledger package keeps its guards: 0 verified guard directive(s), the configuration requires 2
```

**Only verified directives count.** A directive that the scan marked red -
malformed, unknown category, bound to nothing - counts as zero. Otherwise
`//x3:guard` with its body emptied would satisfy the very expectation that
exists to notice its removal: breaking a directive and deleting it check the
same amount, which is none.

**A stale pattern is red, not silent.** If `paths` matches nothing - the
directory was renamed, the files moved - the count is zero and zero meets no
expectation. There is no separate "this pattern is dead" code because none is
needed: the gate is already failing closed.

**The report is not touched.** Expectations are read from it and never written
into it, so the same sources still produce the same bytes and a later
comparison of two runs cannot go red over a clock tick or a configuration
change.

Writing no `expect` section means no expectation and no warning. That is the
right default for a project that has not decided yet, and the wrong one to stay
with: a gate that measures nothing is the failure this engine exists to
prevent.

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

**All nine rule kinds are built.** `deps` with all three of its
matchers — `import` reads the import graph, `literal` reads names the compiler
never sees, `symbol` reads what the code actually uses; `required` asks whether
every file of a class carries a mark; `pairing` asks whether anybody touches a
file at all; `flow` asks where a value may appear; `exposure` asks what reaches
the outside; `duplication` asks whether a body was written twice; `vocabulary` asks which
words a layer must not know; `consistency` asks whether two sets still agree;
`containment` asks whether a component's parts stay under its own root.
The last kind, `containment`, is specified in
[ROADMAP-ARCH.md](ROADMAP-ARCH.md) and have no verifier yet. Naming one in `x3.json` stops the run with exit `2`
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

### Narrowing the source set

`sources` says which files a rule reads. `exclude` takes files back out of it,
and it is read **first**: an exclusion only ever narrows a scope, it can never
widen one. Both live at the top of the `arch` section, where every rule inherits
them, and on a single rule, where the rule's own list **replaces** the inherited
one rather than adding to it — otherwise a rule could never escape a pattern
declared above it.

```json
{
  "arch": {
    "sources": ["**/*.go"],
    "exclude": ["**/*_test.go"],
    "components": { "core": ["internal/core/**"], "modules": ["internal/modules/*/**"] },
    "rules": [
      { "name": "core-must-not-know-modules",
        "kind": "deps", "match": "import",
        "from": "core", "deny": ["modules"] }
    ]
  }
}
```

Tests are the usual reason: a `_test.go` file next to the core is allowed to
reach for a module the core itself must not know. Without an exclusion the only
way to keep that green is to weaken the rule for everybody.

An exclusion is an **escape hatch**, so the engine holds it to the same law it
holds exemptions to:

| Written | Result |
|---|---|
| a pattern that takes at least one file out of `sources` | it applies |
| a pattern that takes **no** file out of `sources` | `dead_exclusion`, red — it excludes nothing, and after the next rename it will go on excluding nothing |
| a pattern that takes **every** file out | `empty_scope`, red — a rule that reads nothing proves nothing |
| `"exclude": []`, or a blank pattern | the run stops with exit `2` before any rule executes |

An empty list is refused rather than ignored because it cannot be told apart
from an absent one: whoever wrote it meant "no exclusions here" and would have
silently inherited the list above instead.

This is not `surface.exclude`, which belongs to an `exposure` rule and drops
paths from the **surface being watched**. `exclude` at the rule level drops
files from what the rule **reads at all**.

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

### The `symbol` matcher — capabilities, not layers

An import ban answers "may this component know that one". It cannot answer "may
this component encrypt, open an outbound request, or retry" — a capability is
usually one call inside a package everybody imports.

```json
{ "name": "modules-carry-no-mechanism",
  "kind": "deps", "match": "symbol",
  "from": "modules",
  "deny": ["crypto/**", "net/http.NewRequest", "**/*.Retry"] }
```

**`deny` holds path patterns here, not component names.** The packages carrying
a forbidden capability are usually not the project's own components, so nothing
is checked against the component list — and a rule that reads no file is still
`empty_scope`.

Two things are read from every file in `from`:

| Read | Written as | Caught by |
|---|---|---|
| each import path | `crypto/sha256` | `crypto/**` |
| each `qualifier.Name` selector | `crypto/sha256.Sum256`, `client.Retry` | `crypto/**`, `**/*.Retry` |

An import alias is resolved to its full path, so `http.NewRequest` is read as
`net/http.NewRequest` whatever the file calls the package. A qualifier that is
not an import — a variable, a receiver — is kept as written, which is what makes
`**/*.Retry` find retry logic whose package cannot be known.

Both are needed. Catching only the import lets a file take a package it already
has and call the forbidden function; catching only the call lets a package be
taken and aliased out of sight.

### `required` — the mark every file of a class must carry

"Every file of this kind carries this protection" is the rule that breaks the
seventh time somebody adds a file. It is one line of configuration instead:

```json
{ "name": "entry-scripts-declare-strict-arguments",
  "kind": "required",
  "sources": ["scripts/**/*.ps1"],
  "marker": "regex:(?m)^\\[CmdletBinding\\(\\)\\]$" }
```

`sources` is the class of file, `marker` is what each of them must carry. The
only marker form today is `regex:<pattern>`, matched against the file's text —
write `(?m)` when the pattern anchors to a line rather than the whole file. The
rule reads whatever the globs name, Go or not.

A finding carries **no line number**: what is missing is missing from the file,
not from one place in it.

### `pairing` — does anybody touch this file?

```json
{ "name": "every-source-file-is-named-by-a-test",
  "kind": "pairing",
  "sources": ["internal/**/*.go"],
  "counterpart": "sibling:*_test.go",
  "requires": "references-a-declaration" }
```

This is **not coverage**. The question is narrower: does any file at all name
this one? Freezing today's state is a `freeze` concern, not this one.

`counterpart` is `sibling:<template>`, and the `*` in the template is the
subject's own base name without its extension — `beta.go` asks for
`beta_test.go`, not for any test file in the directory. The same string read as
a glob says which files are counterparts themselves, and those are never
subjects: asking a test file for its own test is a chain with no end.

| `requires` | Asks |
|---|---|
| `exists` (default) | the counterpart is there |
| `references-a-declaration` | the counterpart also names at least one declaration from the subject |

A subject with no declarations — a file that only carries a package comment —
cannot be asked the second question, so its counterpart only has to exist.

Both kinds name no component, and a configuration with no `components` at all is
valid: only a rule that names one needs the list.

### `flow` — where a value may appear

"Only this package may reach that resource" is usually kept as a list of
packages, and the list goes stale. This asks the narrower question instead:
where may the value itself appear?

```json
{ "name": "restricted-handle-must-not-escape",
  "kind": "flow",
  "sources": ["internal/**/*.go"],
  "value": { "field": "Module.pool" },
  "allow": ["receiver"] }
```

A handle opened with restricted privileges may be a **receiver** and nothing
else: not passed as an argument, not returned, not given another name. There is
no list of who may hold it, because it never leaves.

| Place | The value is |
|---|---|
| `receiver` | the thing a method is called on — `m.pool.Query()` |
| `argument` | passed into a call |
| `result` | returned |
| `assignment` | assigned, declared, or put inside a composite literal |
| `other` | somewhere the engine cannot name — a comparison, an index |

**`allow` lists the permitted places; everything else is red.** A deny list
would leave a place added later silently free, and `other` exists so that an
unrecognised position is refused rather than skipped.

**It reads names, not types.** `m.pool` matches on the field name, whatever `m`
is, and the type in `value.field` is used for one thing: proving the field is
actually declared somewhere the rule reads. If it is not, the rule reports
`empty_scope` — a field that was renamed must not leave a green rule behind.
Once the value is copied into another name, the copy is not followed; that
copy is itself the `assignment` the rule reports.

### `exposure` — what reaches the outside

An internal cost or margin sitting in a struct is nobody's problem. The problem
is the day that struct is written out. Import and flow rules cannot see it: the
field is already in that package, and the violation is that it leaves.

```json
{ "name": "internal-numbers-stay-off-the-tenant-surface",
  "kind": "exposure",
  "surface": { "components": ["core", "modules"], "exclude": ["**/admin/**"] },
  "fields": ["*cost*", "*margin*"],
  "carrier": ["json-tag", "map-key"] }
```

`surface` is where the rule watches — components, minus the paths in `exclude`,
which is how the same field stays legal on an operator screen and illegal on a
tenant one. `fields` are the names that must not reach it, matched
case-insensitively so `unitCost` and `unit_cost` are one rule. `carrier` is how
the name would be written:

| Carrier | Read from |
|---|---|
| `json-tag` | the `json:"…"` name of a struct field |
| `map-key` | a string key in a composite literal |

**A field with no tag is not seen.** Whether an untagged field is serialized at
all needs type resolution, and calling every field an exposure would drown the
gate in noise. What is checked is what the code says it writes out.

### `duplication` — the body written twice

"If a second component has to write the same thing again, it is in the wrong
place." This is the machine form of that sentence.

```json
{ "name": "no-duplicated-mechanism-across-modules",
  "kind": "duplication",
  "across": "modules", "minLines": 8 }
```

Function bodies are reprinted from the syntax tree, then blank lines and
indentation are dropped: comments and formatting differences disappear, and two
copies of the same body match however differently they were laid out. Bodies
shorter than `minLines` are not compared — otherwise two modules both writing
`return nil` would be a finding.

The comparison is between **instances** of one component, so `across` needs a
single star in its patterns; with fewer than two instances present the rule
reports `empty_scope` rather than passing quietly.

**Identifier normalization is off.** With it on, two deliberately separate but
similar bodies would be caught too, and a noisy gate is a gate somebody switches
off. Two bodies are the same only if they are the same after formatting.

### `vocabulary` — the words a layer must not know

An import ban stops the core from calling a module. It does not stop the core
from *knowing* one: a module's name lives in a constant, a field name, a
configuration key, a log line. This is the rule for that.

```json
{ "name": "the-core-speaks-no-module-word",
  "kind": "vocabulary",
  "sources": ["**/*.go", "ui/core/**/*.js", "config/**/*.json"],
  "in": "core",
  "terms": { "componentNames": "modules" } }
```

`in` is the layer that must stay ignorant. `terms` is what it must not know:

| Written | Means |
|---|---|
| `componentNames: "<component>"` | the **instance names** of that component are the terms — no hand-kept list, so a new module is covered the day it appears |
| `words: ["invoice", "billing"]` | terms written out, each one readable word |

The tokenizer is the language gate's, so `alphaTable` is `alpha` + `table` and a
name cannot hide inside camel case. Matching is per word: a term shorter than
three letters is never read, which is why `componentNames` needs a component
whose patterns carry a single star.

**Comments are exempt by default.** Explaining why a rule exists requires naming
the thing; what is forbidden is the *code* knowing it. Set
`"comments": "checked"` when the ban is meant to cover prose as well.

What is read in Go is what the language gate reads: the package name, declared
identifiers, and string literals. A name declared elsewhere is not yours to
spell, and the import rule already guards that boundary. In every other file
type, every line is read.

#### Word forms

A rename is not finished while an inflected form of the old word is still in
the tree, and the list of forms cannot be kept by hand - a language keeps
making them. So a term set can be matched by **form** instead of by whole word:

```json
{ "kind": "vocabulary", "in": "core",
  "terms": { "words": ["invoice"], "match": "forms" } }
```

With `match: "word"` (the default) only `invoice` is a finding. With
`match: "forms"`, any word that *starts with* the term is one - `invoices`,
`invoicing`, `invoice_id` - and the message names the term the form came from.
A term used this way must be at least four characters: a short prefix falls
inside innocent words (`car` would catch `card` and `cargo`), and a noisy gate
is a gate somebody switches off.

Because the extractor reads any text file with a capture group, this kind
reaches past Go. A template calls names that a script has to define, and nobody
compiles either of them:

```json
{ "name": "every-name-a-template-calls-exists",
  "kind": "consistency",
  "left":  { "from": "regex", "sources": ["ui/**/*.html"], "select": "@click=\"([a-zA-Z_][a-zA-Z0-9_]*)\\(" },
  "right": { "from": "regex", "sources": ["ui/**/*.js"],   "select": "function ([a-zA-Z_][a-zA-Z0-9_]*)" },
  "compare": "left-subset-of-right" }
```

The same shape checks a table in a document against reality - a row claiming a
rule "has a gate" on the left, the gate names a script actually runs on the
right.

### `containment` - a component's parts stay under its root

Every part a component owns - its migrations, its scripts, its interface files -
must live under its own root. One part outside, and the component is no longer
movable: deleting it leaves litter, and copying it to another project leaves the
part behind.

The hard question is not where a file is. It is **which component a file belongs
to**, and the answer cannot come from the directory: read that way, every file
is already where it is and the rule would be a tautology that never fires.

So ownership is declared, as a **key** - a short prefix each component puts at
the start of the names of the things it owns:

```json
{ "name": "a-components-parts-stay-under-its-root",
  "kind": "containment",
  "sources": ["**"],
  "keys": { "billing": "blgx_", "orders": "ordx_" } }
```

A path segment that starts with a key marks that file as that component's part -
a file name, a directory name, anywhere in the path. Then the only question left
is whether it sits under one of the component's declared roots:

```
apps/billing/blgx_handler.go          ok
apps/billing/blgx_migrations/1.sql    ok
core/blgx_helper.go                   part_outside_its_root
core/money.go                         carries no key, nobody's part
```

A key must be **5 to 10 characters**. Shorter, and it matches by coincidence -
half the words in a repository contain `bl`. Longer, and it is not a key but a
name, which brings the coincidence back. Two keys may not start alike either, or
one part would answer to two components. All three are configuration errors,
refused before the run starts.

If no file carries any key, the rule is red with `empty_scope`: either the
convention is not in use or a key is misspelled, and both are worth knowing -
a rule that matched nothing has not passed, it simply did not run.


### `consistency` — two sets that must agree

A set of codes produced in code, and a dictionary that gives each of them a
message. When they drift, the user reads a raw key on screen. This is the most
general kind, and the one most likely to produce noise, so the extractors stay
narrow.

```json
{ "name": "error-codes-have-messages",
  "kind": "consistency",
  "sources": ["internal/**/*.go"],
  "left":  { "from": "go",   "select": "const-set:Code" },
  "right": { "from": "json", "file": "i18n/en.json", "select": "keys:error.*" },
  "compare": "left-subset-of-right" }
```

| `from` | Reads | `select` |
|---|---|---|
| `go` | string constants of a named type, in the rule's `sources` | `const-set:<Type>` |
| `json` | the keys of one file, nested keys flattened to `a.b.c` | `keys:<pattern>` |
| `regex` | one capture group, over `file` or `sources` | the pattern itself |

**What enters the set is what was captured**, not the whole key: `keys:error.*`
puts `not_found` into the set, not `error.not_found`, so it can be compared with
the constant that produced it. A `regex` extractor must carry exactly one
capture group for the same reason.

`compare` is `left-subset-of-right` (everything produced has a counterpart) or
`equals` (and nothing is declared that is never produced). Each difference is
one finding, named.

**An empty side is `empty_scope`, not agreement.** A set that could not be read
— a renamed type, a moved dictionary — would otherwise agree with everything.

### Fields a rule has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique in the file; what the report and the stderr lines call this rule |
| `kind` | yes | every kind except `containment`, which stops the run |
| `match` | `deps` only | `import`, `literal` or `symbol` |
| `from` + `deny` | import | the outward question |
| `to` + `allowFrom` | import | the inward question |
| `pattern` + `owner` | literal | only the owner may spell this name |
| `pattern` + `from` | literal | this component may not spell it |
| `from` + `deny` | symbol | this component may not use these names |
| `marker` | required | the mark every file in `sources` must carry |
| `counterpart` + `requires` | pairing | the file that must name this one |
| `value` + `allow` | flow | the value to follow, and the places it may appear |
| `surface` + `fields` + `carrier` | exposure | where to watch, which names, written how |
| `across` + `minLines` | duplication | the component to compare with itself, and the shortest body worth comparing |
| `in` + `terms` + `comments` | vocabulary | the layer, the words it must not know, and whether prose counts |
| `left` + `right` + `compare` | consistency | the two sets and how they must agree |
| `except` | no | `self` only, next to `from` + `deny` |
| `policy` | no | `warn` or `block`; **defaults to `block`**, the same law as live guards |
| `sources` | no | the file set this rule reads; defaults to `arch.sources`, and that to `["**/*.go"]`. The `import` matcher reads Go only; `literal` reads whatever the globs name |
| `exclude` | no | files taken back out of `sources`; defaults to `arch.exclude`. A rule's own list replaces the inherited one. See [Narrowing the source set](#narrowing-the-source-set) |

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
| `forbidden_dependency` | `deps` | a forbidden import edge, or a component spelling or using a name it may not |
| `foreign_resource` | `deps:literal` | a component spelled a name another component owns |
| `escaped_value` | `flow` | the value appeared in a place it may not |
| `exposed_field` | `exposure` | a hidden name reached the surface |
| `duplicate_body` | `duplication` | the same body in two instances of a component |
| `foreign_term` | `vocabulary` | a layer let a word through that it must not know |
| `set_mismatch` | `consistency` | the two sets drifted; each difference is named |
| `missing_marker` | `required` | a file of the class does not carry the mark |
| `missing_counterpart` | `pairing` | no counterpart, or it names nothing from the subject |
| `empty_scope` | every rule | a component the rule names, its own source set, or the field it follows, matched nothing |
| `dead_exemption` | exemptions | an `allow:arch` that no violation needed, or one that binds to no import |
| `dead_exclusion` | `exclude` | a pattern that takes no file out of the rule's sources |

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
| `testdata/symbol-green` with its configuration | `0` |
| `testdata/symbol-red` with its configuration | `1` — four findings: an import path, two calls through it, and a retry |
| `testdata/required-green` / `-red` with theirs | `0` / `1` — a script that lost its mark |
| `testdata/pairing-green` / `-red` with theirs | `0` / `1` — two findings: a file with no counterpart, and one whose counterpart names it nowhere |
| `testdata/flow-green` / `-red` with theirs | `0` / `1` — three findings: the handle as an argument, as a result, and under another name |
| `testdata/exposure-green` / `-red` with theirs | `0` / `1` — the same field is red on the tenant surface and green under the excluded path |
| `testdata/duplication-green` / `-red` with theirs | `0` / `1` — the copy carries an extra comment and an extra blank line, so a run that only compared raw text would miss it |
| `testdata/vocabulary-green` / `-red` with theirs | `0` / `1` — two findings, one in an identifier and one in a `.json` file, while the same word in a comment stays green |
| `testdata/consistency-green` / `-red` with theirs | `0` / `1` — a code the dictionary lost, named in the finding |
| `testdata/exclude` with `arch-exclude-red.json` | `1` — the rule reads the test file and the planted import is red |
| `testdata/exclude` with `arch-exclude-green.json` | `0` — the same tree, with `**/*_test.go` excluded |
| `testdata/exclude` with `arch-exclude-dead.json` | `1` — **two** findings: the pattern that excludes nothing, and the import it therefore failed to hide |
| `testdata/exclude` with `arch-exclude-all.json` | `1` — `**/*.go` empties the rule; both components report `empty_scope` |
| this repository with its own `x3.json` | `0` |
| the same three rules split across three files | `1` — the parts carry the rules |

The Go tests carry the rest of the table: `warn` counts but does not stop the
run, an exemption silences and is listed, a dead exemption is red, an empty
component is red, two runs produce identical bytes, and fifteen broken
configurations are all refused before a rule executes.

A hand-written gate elsewhere is retired only after the `arch` rule has been
seen to go red on the **same** injected violation. Retiring without that double
red is forbidden.

## `x3 freeze`

Some lists are only allowed to get shorter: the exported surface of a core
package, the symbols a binary depends on, the debt somebody promised to pay
down. Hand-written gates for these are always the same three parts — a
measurement, a baseline kept in a file, and a comparison — and the only thing
that differs is what gets measured. The engine carries all three.

```
x3 freeze [-config <file>] [-out <file>] [-update] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or configuration error.

```json
{
  "freeze": {
    "baselines": [
      { "name": "core-surface",
        "sources": ["internal/core/**/*.go"],
        "set": { "from": "go", "select": "exported" },
        "file": "ops/baselines/core-surface.json" }
    ]
  }
}
```

`set` is the same extractor the `consistency` rule uses — `go` (`exported`, or
`const-set:<Type>`), `json` (`keys:<pattern>`), or `regex` with one capture
group — so a baseline can freeze anything a set can be read from. `file` is
where the frozen set lives; the repository keeps it, and a reviewer reads it.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a value that is not frozen | **red** — `baseline_grew`, named |
| a frozen value that is gone | green, counted as **shrunk** |
| nothing measured at all | **red** — `empty_scope` |

`-update` rewrites the baselines to the measured set, and **refuses to write a
set that grew**. That refusal is the gate: an update that accepted growth would
zero the baseline on every run, and no growth would ever be seen again. Shrink
is recorded, because punishing somebody for deleting dead code is how a gate
teaches people to keep it.

A missing baseline file measures against an empty set, so every value is new and
the run is red until `-update` writes the first one. A baseline that cannot be
measured — a moved directory, a renamed type — is `empty_scope` rather than a
quiet pass.

### The control experiment

`check.ps1`, step `freeze control experiment`, runs the binary three times:

| Run | Wants |
|---|---|
| `testdata/surface` against its baseline | `0` |
| `testdata/grown` — one name added to the surface | `1`, the new name reported |
| `-update` on the grown tree | `1`, and the baseline file **unchanged** |

The third row is the one that matters. Without it, a `-update` that quietly
accepted growth would look exactly like a working gate.

### The count mode

Some baselines do not hold a set of values but a **number per key**: how many
lines a document runs, how many files a directory holds, how many places still
reach past a boundary. Freezing the *set* of keys is the wrong gate for those —
it says "this document was already on the list", not "this document grew", and
1471 lines turning into 1499 would pass in silence.

```json
{
  "freeze": {
    "baselines": [
      { "name": "document-length",
        "sources": ["docs/**/*.md"],
        "count": { "of": "lines", "min": 1000, "max": 1500 },
        "file": "ops/baselines/document-length.json" }
    ]
  }
}
```

A baseline writes `set` or `count`, never both. Three measurements are built in:

| `of` | The key | The number |
|---|---|---|
| `lines` | the file | how many lines it has |
| `matches` | the file | how many times `match` occurs in it |
| `files` | the directory | how many files it holds |

`min` is where the gate starts looking: a key at or below it is not measured and
never enters the baseline. Without it the baseline would be a list of every file
in the repository. `max` is a **cap that takes no debt** — a key above it is red
whatever the baseline says, and `-update` leaves it out rather than freezing it.
A cap that could be frozen would be a request, not a limit.

The frozen file carries the numbers, so a reviewer reads the debt instead of
counting it:

```json
{
  "name": "document-length",
  "count": 2,
  "counts": {
    "docs/architecture.md": 1471,
    "docs/protocol.md": 1215
  }
}
```

| Measured against the baseline | Result |
|---|---|
| a key the baseline does not hold | **red** — `baseline_grew` |
| a number above the frozen one | **red** — `count_grew`, both numbers named |
| a number below the frozen one | green, counted as **shrunk**; `-update` records it |
| a number above `max` | **red** — `above_cap`, and never written |
| a key the baseline holds and nothing measures | **red** — `dead_key` |
| nothing measured at all | **red** — `empty_scope` |

The last of those reds is where the two modes part. In the set mode a value
that is gone *is* the shrink — the set is the measurement. Here the measurement
is the number, and a key without one leaves a ceiling standing for a file that
may come back at its old size. `-update` clears the dead keys in the same run
that records the shrinks.

#### The count control experiment

`check.ps1`, step `freeze count control experiment`, runs one baseline over four
trees that differ only in what they measure:

| Run | Wants |
|---|---|
| `count-held` — the numbers the baseline holds | `0`, and the file below `min` not measured at all |
| `count-grown` — one number up, one key gone, one over the cap | `1`, all three named |
| `count-grown`, `-update` | `1`, and the baseline file **unchanged** |
| `count-shrunk` — one number down | `0`, and `-update` writes the smaller number |
| `count-cap`, `-update` | the capped key still **out** of the baseline, the run still `1` |

The fourth row is what makes a baseline shrink at all; the fifth is what keeps
the cap out of reach of the flag that silences everything else.

The `-update` in those rows really runs, so the step writes the baseline files back
afterwards, byte for byte - an experiment that changed what it measures would
be measuring itself by the second run.


## The finding baseline

`freeze` holds a set of values still. Most gates measure something else: they
produce *findings*, and a project that switches one on for the first time meets
a thousand of them in one run. Nobody clears a thousand findings in an
afternoon, so that gate gets switched off again, and a gate that is off measures
nothing. The way out is the one `freeze` already takes, applied one level up:
write down what the tree owes today, and demand that no new debt appear.

```json
{
  "baseline": { "dir": "ops/baselines" }
}
```

The configuration declares a **directory**; the file name comes from the
command, so `x3 comments` reads `ops/baselines/comments.json` and `x3 secrets`
reads `ops/baselines/secrets.json`. That is the rule the cache already follows,
and it means a project turns baselines on once rather than command by command.
Declare nothing and there is no baseline: every finding is red, exactly as in
every version before this one.

| Flag | What it does |
|---|---|
| `-baseline <file>` | read this file instead of the one the configuration derives |
| `-update-baseline` | rewrite the baseline to the findings of this run; growth is never written |

Five commands read a baseline: `comments`, `secrets`, `arch`, `lang` and
`syntax`. Each of them measures the state of a tree, which is where standing
debt lives. `docs` and `scope` read a diff and `boxes` reads a work list — a
finding there describes the change in front of you, not a debt somebody is
paying down, and freezing it would silence the next change instead of the last
one. `freeze` keeps its own baselines, of values rather than findings.

### The identity carries no line number

A baseline keyed by line number moves the day somebody adds an import. The
finding is the same finding, the file shifted under it, and the gate reports a
violation nobody introduced. Two of those and the baseline is refreshed out of
irritation rather than out of work done, which is the same as not having one.

So a finding is identified by **what it is, where it is, and what it says** —
never by where it sits in the file:

| Command | A finding is identified by |
|---|---|
| `comments` | the rule and the file |
| `secrets` | the rule, the file, the pattern and the **masked** sample |
| `arch` | the code, the file, and the rule, subject and object |
| `lang` | the code, the file, and the token with the place it sits in |
| `syntax` | the code, the file and the name of the check |

The identity is stored as a digest, and the entry beside it carries the parts a
reviewer needs to read:

```json
{
  "version": 1,
  "command": "comments",
  "count": 2,
  "findings": [
    { "id": "4ace75caf575", "rule": "block_too_long", "path": "a.go" },
    { "id": "243c6ff993db", "rule": "block_too_long", "path": "b.go" }
  ]
}
```

The digest, not the text, is what the run compares — a baseline that stored the
matched text would put the very value `secrets` masks into a file the repository
keeps.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a finding the baseline holds | green, held, counted in `baselined` |
| a finding the baseline does not hold | **red** — this is the gate |
| a baseline entry the run no longer produces | **red** — `dead_baseline` |

`-update-baseline` writes the measured findings and **refuses to write a set
that grew**, naming every finding that blocked it. Without that refusal the flag
would be a way of turning any red run green, and the baseline would reset itself
on every run.

A missing baseline file measures against an empty set: the run is red and the
first `-update-baseline` writes it. A file that exists and holds nothing is a
different thing — it is a project declaring that it owes nothing — and it can
only shrink. An empty file is a statement, a missing file is a beginning.

### What can never enter a baseline

- **Warnings.** A `warn` finding does not fail the run; it is an observation,
  not a debt. Freezing one would buy nothing and would later turn into a
  `dead_baseline` red the moment somebody fixed it.
- **Scope-integrity findings.** `empty_scope` says the rule measured nothing.
  Freezing it would take a gate that checks nothing and paint it green — which
  is precisely the failure the baseline is supposed to make impossible.
- **Dead markers.** `dead_exemption`, `dead_exclusion` and a parser that is not
  installed belong to the gate's own health, not to the source. A stale
  exemption that could be baselined would never have to be removed.

### The control experiment

`check.ps1`, step `baseline control experiment`, runs one command seven times
over four trees that differ only in the debt they carry:

| Run | Wants |
|---|---|
| `tree`, no baseline yet | `1` |
| `tree`, `-update-baseline` | the baseline file written |
| `tree`, with the baseline | `0` |
| `shifted` — the same debt, moved down the file | `0` |
| `grown` — one file added to the debt | `1`, only the new one named |
| `grown`, `-update-baseline` | `1`, and the baseline file **unchanged** |
| `fixed` — one debt paid, the entry still in the baseline | `1`, `dead_baseline` |

The fourth row is what separates an identity from a line number, and the sixth
is what separates a baseline from a switch that turns the gate off.

## `x3 docs`

Some changes must not travel alone: code without its documentation, a migration
without its release note, a public surface without its changelog line. The rule
is the project's, the question is general — *if you touched here, you touch
there too.*

```
x3 docs [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or configuration error.

```json
{
  "docs": {
    "rules": [
      { "name": "code-changes-carry-documentation",
        "when": ["internal/**", "cmd/**"],
        "then": ["docs/**"] }
    ]
  }
}
```

That is this repository's own section. Its own gate runs this command against
itself on every `check.ps1`.

### What counts as changed

| `-scope` | Reads |
|---|---|
| `auto` (default) | the working tree when it is dirty, the last commit when it is clean |
| `working` | `git status`, including untracked files; a rename counts as its new name |
| `head` | the files in `HEAD`, with the commit body as the place a reason may live |

The default is not a convenience. Checking the last commit while the tree is
dirty would count documentation that has not been written yet — the easiest way
there is to blind this gate.

**A directory that is not a repository is red**, not green: a gate that cannot
read what changed cannot say anything about it.

### Exemption, with a reason

A rule may be skipped by writing its `exempt` marker — `docs: none` unless the
rule says otherwise — followed by an actual reason:

```
docs: none - wording of one stderr line; the capabilities document does not quote it
```

In `head` scope the marker lives in the commit body, where it stays readable
afterwards. In `working` scope it is passed with `-reason`, for the run before
the commit exists.

**The marker alone is red.** `exemption_without_reason` is a separate code from
the missing change itself, because an exemption nobody had to justify becomes
the only path within a month.

### The control experiment

`check.ps1`, step `docs gate`, runs the binary twice: this repository with its
own configuration (`0`), and a rule whose counterpart directory does not exist
(`1`). Without the second run, a gate that silently matched everything would
look exactly like a gate that passes.

The Go tests carry the rest: a code-only commit is red and a code-and-docs
commit is green, a marker without a reason is red and the same marker with one
is green, a dirty tree is read instead of the commit under it, and a directory
with no repository is red.

## `x3 secrets`

A credential in the source is the one mistake that cannot be taken back: it is
in the history, and the history is shared. The patterns ship with the engine and
the project adds its own.

```
x3 secrets [-config <file>] [-out <file>] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or configuration error.
**No `secrets` section is not an error** — the builtin patterns apply. A leak
scan is not a check somebody skips by not configuring it.

```json
{
  "secrets": {
    "sources": ["**"],
    "exclude": ["internal/secrets/testdata/**"],
    "patterns": [
      { "name": "internal-service-token", "match": "svc_[0-9a-f]{32}" }
    ]
  }
}
```

Paths are relative to the directory being scanned. `builtin: false` turns the
shipped patterns off, and then the project must write its own — a scan with no
patterns is refused rather than passed.

### The report carries no secret

**The value found is masked**: the first four characters, then its length.

```
BLOCK config.go:7: secret_found
	a value matching "aws-access-key" is in the source
	value: AKIA... (20 characters)
```

A report is written to a log, pasted into a ticket, shared on a screen. A tool
that found a leak and then printed it would be a second leak. What survives is
enough to recognise the finding and useless to anybody who reads it.

### The shipped patterns

| Name | Catches |
|---|---|
| `private-key-block` | a PEM private key header of any kind |
| `aws-access-key` | an `AKIA`/`ASIA` access key id |
| `google-api-key` | an `AIza` API key |
| `slack-token` | an `xox[baprs]-` token |
| `github-token` | a `gh[pousr]_` token |
| `json-web-token` | a three-part JWT |
| `url-with-password` | a password inside a connection string |

**Every pattern is a recognisable format, not an entropy score.** A value that
looks random is not thereby a secret, and a gate that reds every random-looking
string is switched off within a week. Binary files are skipped for the same
reason: random bytes match anything eventually.

### Excluding what the pattern also catches

A credential format is easy to describe and hard to describe *exactly*. A
pattern for an IPv4 address also matches a private range, a documentation
address reserved by RFC 5737, a browser version string like `126.0.0.0`, and a
date written with dots. Measured on a real production Go application, the bare
patterns for its own credential shapes returned **530 findings where 63 were
real** — and a gate running at eight times noise is switched off within a week.

The exclusion belongs next to the pattern it corrects, one entry per reason:

```json
{
  "secrets": {
    "patterns": [
      { "name": "ipv4-address",
        "match": "(?:^|[^0-9.])((?:[0-9]{1,3}[.]){3}[0-9]{1,3})(?:[^0-9.]|$)",
        "ignore": [
          { "match": "^(?:0|10|127)[.]", "reason": "this host and the private range" },
          { "match": "^(?:192[.]0[.]2|198[.]51[.]100|203[.]0[.]113)[.]", "reason": "RFC 5737 documentation addresses" },
          { "match": "[.]0$", "reason": "a network block or a version string, not a host" },
          { "value": "255.255.255.255", "reason": "the broadcast address" }
        ] }
    ]
  }
}
```

- An entry writes **`match`** (a second expression) or **`value`** (one exact
  value), never both, and always a **`reason`**: an exclusion nobody explained
  is never questioned again. A literal dot reads better as `[.]` than as an
  escape the JSON has to carry twice.
- It reads the **value that was found**, not the line. Excluding by line would
  hide every other value that shares it.
- `"on": "match"` hands it the whole match instead, for the case where the
  surroundings decide rather than the value: an extension is a secret when the
  address around it names a real host, and not when it names a documentation
  one.
- An exclusion that excluded nothing is `dead_ignore` and **red** — the law that
  covers dead exemptions and dead exclusions everywhere else in the engine. A
  stale exclusion is how a gate goes blind quietly.

**A capture group is the value.** A pattern usually has to match the characters
around a value to find it — a separator, a boundary, a prefix — and those
characters are not part of the secret. When the pattern has a capture group, the
mask covers the group and the exclusions read the group; without that rule the
mask sits one character off and every exclusion reads the wrong text. Every
match on a line is examined, not only the first: an excluded value must not hide
the real one beside it.

**Lookaround does not exist here.** Go's regexp engine is RE2, so a pattern
written with `(?<!...)` comes back as `invalid named capture`, which sends the
reader looking for a named group nobody wrote. The engine names the real gap and
points at what replaces it:

```
patterns[0]: match: error parsing regexp: invalid named capture: `(?<![0-9.])[0-9]{15,17}`;
"(?<!" is a lookaround and RE2 has none - write the exclusion as an ignore entry instead
```

Exclusions belong to the pattern that carries them, so the shipped patterns
cannot take one; a project that needs a narrower rule writes its own pattern
with `builtin: false`, or exempts the line where the value sits.

#### The ignore control experiment

`check.ps1`, step `secrets ignore control experiment`, runs the same tree
against two configurations and then two more:

| Run | Wants |
|---|---|
| the noisy tree, pattern with no exclusions | `1`, five findings |
| the same tree, exclusions written | `0` |
| the same tree plus one real address | `1`, that one finding, masked to the capture group |
| an exclusion that excludes nothing in this tree | `1`, `dead_ignore` |
| a pattern written with a lookaround | `2`, a configuration error |

The second row alone would prove nothing — an exclusion that swallowed
everything would also be green. The third is what says the exclusions removed
noise rather than sight.


### Exemption, with a reason

```go
//x3:allow:secret: a documented example key, not a live credential
const example = "AKIAJ4EXAMPLEKEY9ABC"
```

The directive covers **its own line and the one below it**, and it must be the
first thing on its line after the comment opener — `#`, `--`, `/*`, `*`,
`<!--`, `;` — so a sentence that merely *mentions* the directive is not one.
That distinction is not theoretical: this package's own comments describe the
directive, and the first version of the scanner read them as exemptions.

An exemption nobody needed is `dead_exemption` and red, the same law as
everywhere else in the engine — and it applies here too. The example above
carries a real-shaped key on purpose: written with an ellipsis instead, the
exemption over it would cover nothing and this document would fail the scan it
describes.

### The control experiment

`check.ps1`, step `secrets control experiment`, runs the binary five times:

| Run | Wants |
|---|---|
| `testdata/clean` | `0` |
| `testdata/leaky` | `1` — three findings, one of them in a shell script |
| `testdata/exempt` | `0`, and the exemption listed |
| `testdata/dead` | `1` — `dead_exemption` |
| this repository | `0` |

Seeing only the red would not be enough: a scanner that says no to every value
would also exit `1`. The clean and exempted rows are what separate a gate from
a noise generator. The repository's own run is green because three test fixtures
carry reasoned exemptions — which is the feature being used, not worked around.

## `x3 comments`

A comment block that grew past reading length is not documentation, it is a
document in the wrong place. This gate measures two things and treats them
differently on purpose.

```
x3 comments [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [dir]
```

**Block length is red.** Consecutive comment lines form a block; a blank line or
a line of code closes it. Over the limit, the block is a finding:

```
BLOCK internal/source/glob.go:10: block_too_long
        a comment block runs 11 lines, the limit is 10; what needs this many
        lines belongs in a document
```

**The ratio only warns.** When a file carries more comment lines than code
lines, the run says so and stays green. The asymmetry is deliberate: the measure
is *necessity*, not count, and a gate that failed on a ratio would make people
delete comments that were needed. It speaks only above a floor, because a
three-line file with four comment lines is not a finding.

```json
{ "comments": { "block": 10, "doc": 20, "ratio": "warn", "ratioFloor": 30 } }
```

The **opening block** - everything before the first line of code - has its own
limit, twice the ordinary one by default. It is read once and describes the
whole file, so it may say more. "The first block" would have been the wrong
rule: the first block *inside* the code is an ordinary block.

Not every language earns the same opening block. A Go package comment documents
the whole package and is worth its twenty lines; handing the same allowance to
every extension is the same as having no ceiling. A limit can be written per
language, and the general one applies wherever nothing is:

```json
{ "comments": { "doc": 10, "docByExtension": { ".go": 20 } } }
```

Lines that talk to a tool rather than to a reader are neither prose nor code:
`//go:...`, `// Deprecated:`, `//nolint`, `#!`, `// +build`, and x3's own
directives. They close a block and count for nothing.

Ten languages are built in (`.go`, `.js`, `.java`, `.cpp`, `.py`, `.ps1`,
`.yaml`, `.yml`, `.sql`, `.lua`); a file whose extension is not among them is
skipped rather than guessed at. A project adds its own:

```json
{ "comments": { "openers": { ".ts": "//", ".rb": "#" } } }
```

An exemption carries a reason, as everywhere else, and a dead one is red:

```go
//x3:allow:comments: the glob syntax table is the contract itself
```

It may sit **above or below** the block it covers. Above is the natural place,
but `gofmt` moves directives to the end of a Go doc comment, and a rule that
accepted only one side would break itself on the next format.

### The comment control experiment

Four trees: a block inside the limit (green), the same block one line over
(red), the long block with a reasoned exemption (green), and an exemption that
silences nothing (red). The fifth run is this repository itself - the diet is a
house rule here, so the gate that enforces it runs against the house.


## `x3 boxes`

An open-work list is a document, and documents drift: work finishes and the box
stays open, or a box is closed by somebody who meant to finish it. This makes
the list **machine-readable** — each box carries the criteria that would prove
it done — and asks both questions.

```
x3 boxes [-config <file>] [-out <file>] [dir]
```

Exit codes are scan's: `0` green, `1` red, `2` usage or configuration error.

```json
{ "boxes": { "file": "docs/OPEN-WORK.json" } }
```

The list lives in its own file — it changes weekly, while the configuration
changes yearly:

```json
{
  "boxes": [
    { "id": "arch-containment",
      "title": "the containment rule kind, the last of the nine",
      "state": "open",
      "done": [
        { "when": "file", "path": "internal/arch/containment.go" },
        { "when": "pattern", "sources": ["docs/CAPABILITIES.md"], "match": "### `containment`" }
      ] }
  ]
}
```

That is this repository's own list, and `check.ps1` runs this command against
it.

### A box that is not needed yet

Some work is owed only once something else happens: a second tenant, a second
voice application, a version bump that has not landed. Such a box is open and
*should* be, even when the thing that would prove it done happens to exist
already. `when` says what makes it due:

```json
{ "id": "second-voice-application",
  "title": "the billing migration a second voice application would need",
  "state": "open",
  "when": [{ "when": "pattern", "sources": ["apps/*/kind.json"], "match": "\"voice\"" }],
  "done": [{ "when": "file", "path": "docs/migration-b.md" }] }
```

While the condition does not hold, the box waits: it stays open without being
red, and the run counts it. The moment the condition holds, the box is measured
like any other - if the work is already done, leaving it open is red.

The rule that does **not** relax is the other one: a conditional box still
cannot be closed without proof. A box closes because the work was done, never
because it stopped being needed.

### Both directions, or neither

| State | Criteria | Result |
|---|---|---|
| `open` | all met | **red** — `box_finished`: the work is done, the list is stale |
| `done` | any unmet | **red** — `box_unproven`, each unmet criterion named |
| `open` | some unmet | green |
| `done` | all met | green |

Asking only the second question lets a list fill up with finished work. Asking
only the first leaves closing without proof free. The value is in asking both.

### The three criteria

| `when` | Holds when |
|---|---|
| `file` | `path` exists |
| `pattern` | `match` is found in any file under `sources` |
| `absent` | `match` is found **nowhere** under `sources` |

`absent` is the one that makes deletion provable: "the old call site is gone"
is exactly the sentence that becomes true when that work finishes, and nothing
else in the engine can state it.

A list with no boxes is `empty_scope`. A list that says nothing does not say
everything is finished.

### The control experiment

`check.ps1`, step `boxes control experiment`, runs the binary four times against
one tree: a list that matches it (`0`), a list that leaves finished work open
(`1`), a list that closes unfinished work (`1`), and this repository's own list
(`0`).

## `x3 syntax`

A compiler tells you when a file does not parse. Nobody compiles a template, a
settings file, or a script the browser will read at run time - so those go out
broken, the server still answers 200, and the screen is simply blank.

```
x3 syntax [-config <file>] [-out <file>] [dir]
```

The gate does not guess which parser a file wants; a project declares it. There
are three kinds of check, and each check is exactly one of them:

```json
{
  "syntax": {
    "checks": [
      { "name": "every-settings-file-parses",
        "sources": ["**/*.json"], "as": "json" },

      { "name": "browser-scripts-parse",
        "sources": ["ui/**/*.js"], "run": ["node", "--check"] },

      { "name": "no-escaped-quote-in-an-attribute",
        "sources": ["ui/**/*.html"], "deny": "=\"[^\"]*\\\\'",
        "reason": "a backslash escape inside an attribute is not valid here" }
    ]
  }
}
```

`as` names a parser the engine carries - `json` is the only one, because a
format half-understood is worse than a format not understood at all. `run` names
an external parser: the file path is appended to the command, and a non-zero
exit is a finding with the parser's own first line of output. `deny` is the
other half of the same problem - text that parses but means nothing in this
format, and it needs a reason, like every other silence-or-refusal in the
engine.

**A parser that is not installed is red.** That is the default, and it is the
point: a gate that quietly skips its check on a machine without the tool is a
gate that reports green having verified nothing. A project that genuinely wants
the check optional says so:

```json
{ "name": "browser-scripts-parse", "sources": ["ui/**/*.js"],
  "run": ["node", "--check"], "missing": "warn" }
```

Files are independent, so each check runs its files on every core. A check whose
sources match nothing is red with `empty_scope`, for the reason every other gate
here has that rule.

### The syntax control experiment

Four runs. A tree whose files parse (green), the same tree with one file broken
(red), a check naming a parser that is not installed (red), and the same check
with `missing: "warn"` (green, and the skip is still printed). The third and
fourth are the pair that matters: without them, a missing tool would look
exactly like a clean run.

## `x3 scope`

The other half of the coupled-change problem. `x3 docs` asks what a change must
bring **with** it; this one asks what a change must **stay away from**. A part
that ships on its own stops shipping on its own the day it rides in the same
commit as something else.

```
x3 scope [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

A lane is a set of paths, plus the paths that are allowed to travel with them:

```json
{
  "scope": {
    "lanes": [
      { "name": "site",
        "paths": ["services/site/**"],
        "also":  ["docs/site/**"],
        "exempt": "lane: crossed" }
    ]
  }
}
```

A change that touches nothing in `paths` is none of this lane's business. A
change that does touch it must stay inside `paths` and `also`; anything else is
named, file by file:

```
BLOCK site: outside_the_lane
        this change is in the "site" lane (1 file(s)) and also touches 1 file(s)
        outside it; split the change, or say "lane: crossed" followed by a reason
        outside: internal/core/money.go
```

Crossing a lane is allowed when it is said out loud. The exemption goes in the
commit message (or `-reason` for a working-tree run), and a marker with nothing
after it is red - the same rule `docs` has, for the same reason: a marker
anybody can type without saying why is a marker everybody types.

`-scope auto` reads the working tree when it is dirty and `HEAD` when it is
clean, so a developer before the commit and a gate after it type the same
command. Both this gate and `x3 docs` read the diff through the same code
(`internal/changed`), because two gates that disagreed about what a commit
touched would each be right about a different commit.

### The lane control experiment

A temporary repository and three commits: one inside the lane (green), one that
touches the lane and a file outside it (red, naming the file), and one that
crosses with a reason in the message (green, counted as an exemption). The
middle one is what the gate is for; the outer two are what keep it from being a
gate that refuses everything.

## `x3 record`

Every other checker here reads the project **at rest**: files, imports, names,
sets. This one reads it **in motion**. x3 stands in front of the running
application as a reverse proxy, passes the traffic through untouched, and
writes down what went by.

```
x3 record [-config <file>] -listen <addr> -target <url> -ledger <file>
```

```
x3 record -listen :9100 -target http://localhost:8080 -ledger x3/ledger/api.jsonl
```

The application is not modified, not rebuilt and not linked against x3 — no
middleware, no import, no build tag. That follows the standing rule that a
project never carries a bridge script for the engine, and it makes the
capability language-independent from the first line: the recorder does not know
or care what the application behind it is written in.

The command listens until it is interrupted, then reports how many interactions
it wrote. Exit codes: `0` when it shut down cleanly, `2` for usage,
configuration or I/O errors. There is no `1` — recording is not a gate. It
produces the source a later run is compared against, and that comparison
(`x3 replay`) is **not built yet**; the specification it will be measured
against is `docs/ROADMAP-X4.md`.

### The ledger

One file per suite, JSON Lines, one interaction per line:

```json
{"n":1,"req":{"method":"POST","path":"/orders","query":{},"headers":{"Content-Type":["application/json"]},"body":{"name":"a cup"}},"res":{"status":201,"headers":{"Content-Type":["application/json"]},"body":{"id":"17","state":"created"}},"ms":34}
```

One line per interaction is deliberate: a behaviour change then shows up as a
**diff a human can read** in review, which a single re-serialised document would
not. `n` is the recorded order, and replay will follow it. Header and query
values are kept as lists, because a header folded into one string comes back
different when it is sent again.

A JSON body is stored parsed, so a change inside it reads as one changed field
rather than one changed string; anything else is stored as text. `ms` is written
for the reader — nothing compares it.

The ledger is a source file: it is committed, it is reviewed, and it is the
thing that shrinks a pile of hand-written behaviour tests. Which is exactly why
nothing secret may reach it.

### Redaction happens before the disk

A secret that was never written cannot leak from a ledger later, so redaction
sits between reading the response and writing the line — not in a cleanup pass
afterwards. Three layers run over every interaction, and the first two need no
configuration at all:

1. **Credential headers**, always: `Authorization`, `Cookie`, `Set-Cookie`,
   `Proxy-Authorization`. What they carry is identity, not behaviour.
2. **The `secrets` pattern set** — the same patterns `x3 secrets` searches the
   source with, applied to every recorded value.
3. **The project's own field paths**, declared under `record.redact`.

```json
{
  "record": {
    "redact": [
      { "path": "res.body.token",         "reason": "session token" },
      { "path": "res.body.items.*.email", "reason": "personal data" },
      { "path": "req.headers.X-Api-Key",  "reason": "customer key" }
    ]
  }
}
```

A rule without a reason is refused, in the house style: whoever reads the ledger
sees *why* a field is hidden next to the fact that it is hidden.

A path starts with `req` or `res`, then `headers`, `query` or `body`; the rest
walks into the body, where `*` means every element of an array or every field of
an object. A path that reaches nothing is not an error — that field simply did
not appear in this run. A path that cannot mean anything (`res.query.page`, a
`headers` without a name) is a configuration error, because a misspelled rule
would otherwise look exactly like a rule that had nothing to hide.

Hidden values are written as `"<redacted:reason>"`, so a reader can tell a masked
field from an absent one:

```json
{"headers":{"Authorization":["<redacted:credential-header>"]},
 "body":{"key":"<redacted:secret:aws-access-key>","name":"a cup"}}
```

The proxy itself stays transparent: the client receives the application's answer
exactly as it was sent, cookies and all. Redaction applies to what is written
down, never to what is served.

### What is recorded, and what is not

**Inbound HTTP**, by decision. In-process middleware would require the project
to import x3 and tie the capability to one language; recording what the
application asks of *other* services needs a stand-in for the far side;
function-level capture needs instrumentation. Each is a later version rather
than a v0 shortcut, and each is a box in `docs/OPEN-WORK.json`.

Connection headers (`Connection`, `Keep-Alive`, `Transfer-Encoding` and the
rest) are neither forwarded nor recorded: the ledger holds the request as it was
*forwarded*, so replaying it cannot send a proxy's own connection settings on to
the application.

When the target does not answer, the client is told so with `502` and **no line
is written**. There is no behaviour to record — that answer came from the proxy,
not from the application.

### The control experiment

`check.ps1` starts a small application (`internal/record/testdata/echo`), puts
the recorder in front of it, sends real traffic through, and then asks four
questions of the ledger: the credential header is hidden, a planted key in the
shape of a real one is hidden, the raw key appears nowhere in the file, and an
ordinary undeclared field is **still there**. The last one is the half that is
easy to skip — without it, a recorder that masked every field would pass just as
well.

## `x3 replay`

`record` writes a run down; this one sends it again and compares. The behaviour
test is not a file somebody wrote — it is a recording the machine took.

```
x3 replay [-config <file>] -target <url> -ledger <file> [-out <file>]
```

```
x3 replay -target http://localhost:8080 -ledger x3/ledger/api.jsonl
```

The recorded requests go out in the recorded order against a freshly started
application, and each answer is compared with the one on file. Exit codes are
the usual: `0` green, `1` red, `2` usage, configuration or I/O error.

Three things are compared: the **status code**, the **headers named in
configuration**, and the **body, field by field**. Every field that is not named
in a rule is compared exactly — the default is fail-closed, the direction every
other gate here points.

```
DIFF 2 res.body.state: value_differs
        the value is not the one that was recorded
        recorded: created
        received: queued
```

### What is allowed to differ

A recording that compares timestamps fails on the second run. `normalize` names
the fields that may differ, and how:

```json
{
  "replay": {
    "headers": ["Content-Type"],
    "normalize": [
      { "path": "res.body.created_at", "as": "time" },
      { "path": "res.body.id",         "as": "uuid" },
      { "path": "res.body.items.*.n",  "as": "number" },
      { "path": "res.headers.Date",    "as": "any" }
    ]
  }
}
```

`time`, `uuid`, `number` and `any` are the four kinds. A normalized field is not
compared by value — but its **presence and kind still are**. Dropping the field
entirely, or returning a string where a time was recorded, is a difference:

```
DIFF 2 res.body.created_at: kind_differs
        the answer is not a time any more
```

`headers` lists the response headers that take part; it defaults to
`Content-Type`, because the shape of a body is behaviour while `Date` and
`Content-Length` are not. This is the one place where the comparison is opt-in
rather than fail-closed, and the reason is that a full header comparison is red
on every run — a gate that is always red is a gate somebody switches off.

### Exemptions, and the dead ones

An exemption is declared in `x3.json`, never inside the ledger, always with a
reason — and an exemption that silenced nothing is **red**:

```json
{ "replay": { "ignore": [ { "path": "res.headers.X-Request-Id",
                            "reason": "per-request id, not behaviour" } ] } }
```

```
DIFF res.headers.X-Request-Id: dead_exemption
        no difference needed this exemption
```

A stale exemption is how a gate goes quietly blind, so it is treated the same
way `secrets` treats one. `normalize` rules are not held to this: a rule for a
field that did not appear in this run says nothing about whether the rule is
still needed.

Paths are the ones `record.redact` uses — `req`/`res`, then `headers`, `query`
or `body`, then into the body, where `*` matches one segment (an array element
or a field name).

### Carrying a session

Recording masks credentials, so a suite behind a login cannot simply be sent
again: the `Authorization` header on file says `<redacted:credential-header>`,
and sending that back would come home as `401`. The way out is not to unmask the
recording - it is to take a **fresh** value from the run itself:

```json
{ "replay": { "carry": [
    { "from": "res.body.token",
      "into": "req.headers.Authorization",
      "as":   "Bearer {value}" } ] } }
```

Every answer is read for `from`; whatever it yields is poured into `into` on the
requests that follow, through the `as` template. The login in the recording
issues a new token on replay, and the requests after it carry that one. A value
may also come from the environment - `"from": "env:X3_TOKEN"` - for a credential
no answer contains.

A value is carried **into a request only**: writing into a response would mean
editing the thing being compared. The template must have a `{value}` in it, and
a `from` that is neither a path nor `env:` is a configuration error.

If the field is not in the recorded request at all, it is added. That is the
common case: the header was masked away, and what replaces it is not the old
value but a live one.

### Replaying in parallel

Requests go out one at a time in the recorded order by default. Where the
interactions are independent, `workers` sends them on several connections at
once:

```json
{ "replay": { "workers": 8 } }
```

Measured against a target that waits 20 ms per call, the way a real service
does - 60 interactions:

| Run | Time |
|---|---|
| sequential | 1.28 s |
| 8 workers | **0.21 s** |

Against a local application answering instantly the two are the same, because
what parallelism buys is the waiting, not the work. The report is identical
either way: findings are collected in interaction order, so the same ledger
produces the same bytes whatever the worker count.

Parallelism is declared, never assumed. Only the project knows whether its
recorded interactions are independent - two orders posted at once are, a login
and the request after it are not. **`workers` and `carry` together are a
configuration error**, refused before the run: a carried session needs the
recorded order, and going faster while getting a different answer is not going
faster.

### A database of its own

`testdb` and `replay` need no new feature to pair; the existing commands
compose:

```
x3 testdb run -- ./start-app-and-replay.sh
```

`testdb run` clones a template database for the run, exports its DSN as
`X3_TESTDB_DSN`, runs the command, and drops the database afterwards. The
script starts the application against that DSN and calls `x3 replay`. What the
engine deliberately does not do is start the application itself - it does not
know how, and a wrong guess would be worse than the two lines of script.

### What a recording cannot send back

A masked value is not sent to the application. `<redacted:credential-header>` as
an `Authorization` header would arrive as a real credential and come back `401`,
and a reader would file an identity error as a behaviour change. Those values
are counted instead, and the count is on the last line of every run:

```
x3 replay: 2 interaction(s) - 0 difference(s) - 0 exempted - 1 value(s) could not be sent back
```

This is the honest edge of v0: a suite behind authentication is recorded fine,
and replays as an unauthenticated one. Sessions — a login whose token the
following requests carry — are a box in `docs/OPEN-WORK.json`, not a promise
made here.

The report is JSON, like every other command's, and carries no timestamp: the
same ledger against the same application must produce the same bytes. Recorded
and received values are truncated and run through the `secrets` pattern set
before they are printed — the report that finds a leak must not become one.

### Calls the application makes

`x3 record` sees what the world asks of the application. This sees what the
application asks of the world - the rate service, the mail gateway, the payment
provider - and later answers those calls itself, so a replay does not reach
anybody outside.

```
x3 outbound record -listen :9101 -ledger x3/ledger/out.jsonl
x3 outbound serve  -listen :9101 -ledger x3/ledger/out.jsonl
```

This one is a **forward** proxy, not a reverse one: the application is told
about it the way every HTTP client already understands, with `HTTP_PROXY`. Once
again nothing is imported and no code changes.

In `record` mode the call goes out, comes back, and is written down with the
same redaction the inbound ledger gets. In `serve` mode nothing goes out at all:
the answer comes from the ledger, matched on method plus scheme, host and path,
in recorded order - so an application that calls the same endpoint twice gets
the first answer first.

A call the ledger never saw is refused with `502` and counted. That is the point
of the mode: during a replay, a *new* outbound call is new behaviour, and a
proxy that quietly let it through would hide exactly what the replay is for. The
command exits `1` when the count is above zero.

**Encrypted calls are refused, not tunnelled.** A `CONNECT` gets `501` and a
line on stderr. Recording HTTPS would mean terminating TLS with a certificate of
x3's own, and believing you recorded a call you did not is worse than knowing
you did not record it.

### The control experiment

`check.ps1` records two requests against a small application
(`internal/record/testdata/echo`) whose answers carry a fresh id and timestamp
every time, and then replays the same ledger three times:

- **without a `normalize` rule** — red, because the id and the timestamp differ;
- **with the rule** — green, on the very same ledger and the very same run;
- **against the application started with `-drift`**, which answers `queued`
  where it answered `created` — red, and the report names `res.body.state`.

The first run is the one that is easy to leave out, and it is the one that
proves the rule is doing something. The third proves the gate can still see a
real change while the rule is in force.

## `x3 guard`

```
x3 guard [-config <file>] [-report <file>] [-only <tags>] [-skip <tags>] [-stamp] -- <command> [args...]
```

Runs the **live guards** declared in a configuration file, then decides whether
the command after `--` may start. This is the *guard-then-launch* shape: one
process, one decision, no wrapper script.

| Part | Meaning |
|---|---|
| `-config <file>` | configuration file holding the guards; defaults to `x3.json` |
| `-report <file>` | write the JSON report here. **Without it no report is written** — stdout belongs to the launched command |
| `-only <tags>` | run only the guards carrying one of these comma separated tags. See [Choosing which guards run](#choosing-which-guards-run) |
| `-skip <tags>` | skip the guards carrying one of these comma separated tags |
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

### Choosing which guards run

One configuration file usually holds every live guard a project has, but not
every gate needs all of them: the one that starts a worker has no business
waiting on the guard that belongs to a different binary. `tags` on a guard and
`-only` / `-skip` on the command pick a subset **out of the same file**, so a
narrower run is still the file everybody reviews rather than a second copy that
drifts.

```json
{ "name": "database-reachable", "kind": "sql", "tags": ["db", "slow"],
  "dsnEnv": "APP_DSN", "query": "select 1", "equals": "1" }
```

```
x3 guard -only db   -- ./worker      # the database guards, and every untagged one
x3 guard -skip slow -- go test ./...  # everything except the slow ones
```

| Written | What runs |
|---|---|
| neither flag | **every guard in the file** — the behaviour a project already had, unchanged |
| `-only a,b` | guards carrying `a` or `b`, **plus every guard with no tags at all** |
| `-skip a` | everything except the guards carrying `a` |
| both | `-skip` wins on a guard that matches both |

**A guard with no tags always runs.** Narrowing a set must not drop the check
nobody got round to classifying; that is the same fail-closed reading the engine
gives an unwritten `policy`. It also means `-only` narrows only *among tagged
guards* — to run exactly one guard and nothing else, every guard in the file
needs a tag.

Three ways to write a selection are refused outright, with exit `2` and before
any guard runs:

| Written | Why it is refused |
|---|---|
| a tag no guard carries | a misspelled `-skip` would otherwise skip nothing and read as if it had — and a misspelled `-only` would quietly run the wrong set |
| a selection that leaves no guard | an empty run is a silent pass, the same reason an empty rule list is an error |
| `"tags": [""]` on a guard | an empty tag selects nothing |

Whatever a selection dropped is named in the report's `skipped` list and in the
stderr summary (`2 guard(s), 1 skipped`). A check that did not run must never
look like a check that passed.

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
| `tags` | no | names `-only` and `-skip` select on; a guard with none always runs. See [Choosing which guards run](#choosing-which-guards-run) |
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
| `skipped` | the guards a `-only` / `-skip` selection left out, by name; absent when nothing was dropped. A check that did not run must not look like one that passed |
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

Selection gets its own step, and it is the sharper experiment: **one** file,
`guard-tagged.json`, holding an untagged guard, a `fast` one and a red `slow`
one. Only the flags change, so anything that moves is the selection's doing:

```
== guard selection control experiment
  no selection: exit=1 ran=False (want 1/False - every guard runs, the red one decides)
  -skip slow:   exit=0 ran=True (want 0/True)
  -only fast:   exit=0 ran=True (want 0/True)
  mistyped tag: exit=2 ran=False (want 2/False)
```

The first line is the one that matters to a project already using `x3 guard`:
with no flags all three guards run and the decision is what it always was. The
last line is the one that matters to the gate: a tag nobody carries stops the
run instead of skipping nothing, so `-skip` can never be the quiet way to turn a
red gate green.

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

## `x3 update`

```
x3 update [-config <file>] [-version <tag>] [-source <address|dir|owner/repo>] [-check]
```

The binary replaces itself with a published one, after verifying its SHA256.
This exists so that a project using x3 does not have to carry a downloader of
its own: reading the release layout, checking the sum and putting the new file
in place are the engine's job, and a project that repeats them writes a script
that only its author can vouch for.

The order is fixed:

1. resolve the tag - `-version` if given, otherwise the one named in the
   release pointer;
2. find the checksum this platform's binary must have — from
   [`update.pin`](#updatepin--the-checksum-the-project-itself-vouches-for) when
   the project wrote one, otherwise from the release's `SHA256SUMS.txt`;
3. download the binary;
4. compute its SHA256 and compare it with the expected one;
5. put it in place of the running file.

**A sum that does not match stops at step 4 and the running binary is left
exactly as it was.** The same is true when the release lists no binary for this
platform: a missing file is treated like a wrong one. Both exit `1` and name the
file that was left in place. Not being able to *reach* the source is a
different answer - that exits `2`, because "the release refused me" and "the
network refused me" are not the same event and must not wear the same colour.

**The layout.** Every source, remote or local, is read the same way:

```
<source>/<ref>/<file>
```

- `<source>/main/LATEST` - one line, the newest release tag
- `<source>/<tag>/SHA256SUMS.txt` - the checksum list, in `sha256sum -c` format
- `<source>/<tag>/x3-<goos>-<goarch>[.exe]` - the binary for one platform

`LATEST` is written by the same run that builds the binaries. A separate step
would drift, and a pointer naming a release nobody published is the quietest
way to break an update.

**Where it downloads from**, first answer wins: `-source`, then the
`X3_UPDATE_SOURCE` environment variable, then `update.source` in the
configuration, then the engine's own public repository. The order runs from
outside in on purpose - pointing one run at a mirror should not require editing
a tracked file.

A source may be an address (`https://...`), an `owner/repository` shorthand
(read from `raw.githubusercontent.com`), or **a local directory**. The
directory case is not a test fixture: an air-gapped or mirrored environment
publishes the same three files onto a share and every machine updates from it
with no code path of its own. An existing directory wins over the shorthand
reading - if you have a folder called `mirror/x3`, you meant the folder.

**`-check` changes nothing.** It prints the tag the source publishes and exits
`1` if that tag is newer than the running binary, `0` if it is not. Use it in a
gate that wants to report drift without installing anything.

Output is split so a script can read it: the tag goes to stdout on a line of
its own, everything written for a human goes to stderr.

**Replacing a running file.** The new bytes are written next to the target
first, because a rename is only atomic within one filesystem. On Windows a
running executable cannot be overwritten but can be renamed, so the sequence is
three steps - write, move the old one aside, put the new one in place - and if
the last step fails the old name is given back. The old file is then deleted; if
the running process still holds it, it stays and the next update removes it,
which is not a reason to call a finished update red.

Configuration is optional. The section is:

```json
{
  "update": {
    "source": "owner/repository",
    "latestRef": "main",
    "timeoutMs": 120000
  }
}
```

An unknown key in it is an error, not a silent skip.

### `update.pin` — the checksum the project itself vouches for

Step 4 above compares the download against `SHA256SUMS.txt`, and that list ships
**inside the release it describes**. It catches a truncated download, a mirror
that fell behind, a corrupted file. It cannot catch a compromised release:
whoever can replace the binary can replace the list beside it, and the update
goes green. A release that vouches for itself is not a supply chain guarantee.

`update.pin` moves the expected checksum into the project's own repository,
where it is reviewed, versioned and diffed like any other line:

```json
{
  "update": {
    "pin": {
      "v0.33.0": {
        "x3-windows-amd64.exe": "e2c2bd46...",
        "x3-linux-amd64":       "b21f4cc9..."
      }
    }
  }
}
```

**When a pin is written, `SHA256SUMS.txt` is not read at all.** There is nothing
for it to add: the only checksum that binds anything is the one the project
already agreed to. The stderr line says which authority it obeyed, so a run
never leaves that ambiguous:

```
x3 update: v0.32.0 -> v0.33.0 (x3-linux-amd64, 12905472 bytes, sha256 b21f4cc9..., verified against update.pin)
```

The pin is keyed by **release tag**, not by binary name alone. That is what lets
the engine tell two different refusals apart — "these are not the bytes I
pinned" and "this is a release I never pinned" are different events, and a flat
list would have reported the second as a checksum mismatch and sent the reader
looking for corruption that was not there:

| Situation | Result |
|---|---|
| the tag is pinned and the bytes match | installed |
| the tag is pinned and the bytes differ | exit `1`, **the running binary is left in place**, and the message names `update.pin` as the source of the expectation |
| the tag is not in `pin` | exit `1`, naming the tags that *are* pinned. Upgrading is a deliberate act: pin the release, then install it |
| the tag is pinned but not for this platform | exit `1` — a machine whose binary nobody pinned gets no weaker guarantee than the others |
| no `pin` at all | unchanged: `SHA256SUMS.txt` decides, exactly as before |

Because an unpinned tag is refused, a pinned project does not follow `LATEST` by
accident. `x3 update` with no `-version` resolves the newest tag, finds it
unpinned and stops — which is the point. A new release enters the project the
day somebody writes its checksum down.

A malformed pin is a **configuration error** (exit `2`) rather than a mismatch:
an empty `pin`, a key that is not a release tag, a tag pinning no binary, an
empty binary name, or a checksum that is not 64 hexadecimal characters. Reported
as a mismatch, a mistyped checksum would leave a project unable to update and
unable to see why. Upper case is accepted and lowered — `sha256sum` writes lower
case, but not every tool does.

The checksums to write are the ones a release run prints, and they are also in
the published `SHA256SUMS.txt`; copying them from there is fine, since the
question a pin answers is not "were these bytes ever right" but "are these still
the bytes we reviewed".

### The minimum version gate

A project can state the oldest engine it is willing to be checked by:

```json
{
  "x3": { "min_version": "v0.30.0" }
}
```

This runs **before every command**. If the binary's own tag is older, the
command does not start:

```
x3: RED - this binary is v0.29.0, the project requires v0.30.0 or newer
	run: x3 update
```

`x3 update` is the one command exempt from the gate - it is the answer the gate
points at, and a red with no way out is a wall, not a gate.

Details that matter:

- **An untagged binary satisfies nothing.** A build produced outside a release
  prints `unreleased`, and the gate treats that as failing any requirement. A
  binary nobody published cannot prove which code it carries.
- **A `git describe` suffix is ignored.** `v0.30.0-3-gabc1234` and
  `v0.30.0-dirty` both count as `v0.30.0`: those commits come *after* the tag,
  so such a binary is at least as new as the tag it names.
- **A requirement that is written must parse.** A missing file, a missing
  section or an empty field means no requirement and no warning. A value that
  is not a release tag, or an unknown key beside it, is an error - a misspelled
  requirement that is silently ignored leaves its author believing a gate is
  running.
- **It reads the configuration the way every other section is read**, so the
  requirement may live in an included part rather than in the root file.

### The update control experiment

`check.ps1`, step `update control experiment`, publishes two local releases into
a temporary directory — one whose `SHA256SUMS.txt` is correct, one whose entry
is deliberately wrong — and updates a binary built as `v0.0.1`. The claim
"it was replaced" is never taken from a filename: it is read back out of the
binary by running `x3 version` on it afterwards.

```
== update control experiment
  installed:         exit=0 now v9.9.9 (want 0 / v9.9.9)
  planted checksum:  exit=1 still v0.0.1 (want 1 / v0.0.1)
  below min_version: exit=1 (want 1)
  meets min_version: exit=0 (want 0)
  pin matches:       exit=0 now v9.9.9 (want 0 / v9.9.9)
  pin disagrees with a SOUND release: exit=1 still v0.0.1 (want 1 / v0.0.1)
  tag not pinned:    exit=1 still v0.0.1 (want 1 / v0.0.1)
```

The sixth line is the one that earns `update.pin` its place. The release it runs
against is the **good** one — binary and `SHA256SUMS.txt` agree perfectly, which
is also exactly how a compromised release looks. Only the project's own pin
disagrees, and the update is refused. Without that direction the pin could have
been doing nothing but repeating what the release already said.

`internal/release/release_test.go` carries the rest: a pin that matches installs
and reports `update.pin` as the authority, an unpinned tag and an unpinned
platform are refused separately, and five malformed pins are all rejected when
the configuration is read rather than when the download is compared.

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

## Speed

A gate that takes a minute is a gate somebody stops running. Files are
independent of each other, so reading and parsing them is done on every core and
the results are put back in file order — the report is byte-identical whatever
the core count.

Measured on a real Go application of **1174 Go files** (2040 files in total),
sixteen cores:

| Command | Before | After |
|---|---|---|
| `x3 lang` | 5.09 s | **0.24 s** |
| `x3 scan` | — | **0.12 s** |
| `x3 secrets` (all 2040 files) | — | **0.54 s** |

The same run produced the same bytes before and after the change, which is the
part worth checking: a parallel walk that reordered its findings would turn
every later comparison into noise.

### The incremental cache

Since v0.20.0 a run can remember what it measured. The cache is keyed on the
**content** of each file, so a file that did not change is not measured again —
and one that did is measured whatever the cache says.

```json
{ "cache": { "dir": ".x3cache" } }
```

A directory, not a file: the file name comes from the command, because two
commands sharing one file would each delete the other's entries on every run.
Nothing is written unless the section is there — a tool does not leave files on
disk uninvited — and the directory belongs in `.gitignore`, since a cache is
something a machine can rebuild. `x3 scan`, `x3 lang` and `x3 secrets` read it;
`-cache <file>` points one run somewhere else, and `-no-cache` measures
everything again.

An entry is used only when three things match: the **engine version**, a
**fingerprint of the whole configuration**, and the file's **content hash**. Any
of them changing empties the cache. Guessing which configuration section affects
which checker would be cheaper and would eventually be wrong; measuring again is
never wrong.

Measured on the same 1174-file application, second run against the first:

| Command | Full scan | Cached | Cache size |
|---|---|---|---|
| `x3 secrets` (2040 files) | 0.34 s | **0.12 s** | 0.5 MB |
| `x3 scan` (1174 Go files) | 0.13 s | **0.09 s** | 145 KB |
| `x3 lang` (1174 Go files) | 0.17 s | 0.18 s | 5.8 MB |

`lang` is the honest row: on that application the language gate is red on
thousands of lines, and reading 5.8 MB of stored findings costs as much as
parsing the files again. **The cache pays off when a checker's output is much
smaller than its input** — which is why it is declared per project rather than
switched on for everybody.

Every one of those runs produced a report byte-identical to the uncached one.
That is the property the cache is worth having only if it holds.

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

**Pin a version, and let the engine fetch itself.** The consuming project
writes the version it requires into its own `x3.json`
([the minimum version gate](#the-minimum-version-gate)) and calls
[`x3 update`](#x3-update) to obtain that binary. Nothing else about x3 is
tracked in the project: no downloader, no checksum file, no path.

This corrects earlier advice, and the reason is worth keeping. The first
integration had the project carry its own script to read a pinned version,
download the binary and verify the sum. That script was correct and it was
still wrong: every project using the engine would write the same one, each with
its own bugs, and the engine could fix none of them. Fetching and verifying a
release is the engine's own subject. A checked-in path (or an environment
variable holding one) is worse still - green on the machine that wrote it,
unmeasured everywhere else, and unable to say *which* build ran.

**A missing or mismatched binary is red, not skipped.** The pilot's gate was
fail-open at first: no binary meant a warning and a normal start. That is the
failure this engine exists to prevent, so it now refuses — no binary, wrong
version, wrong checksum, all three stop the run and print the command that
fetches the pinned release. "The tool was not there" and "the tool found
nothing" must never produce the same colour.

**The counts are the engine's job, not yours.** This was measured on the pilot:
a scan of a tree with no directives in it exits `0` and reports `0 red`, so a
gate that trusts the exit code alone turns "delete the directives" into a way to
go green. The first answer was to have the consuming project read the report and
assert on it - which worked, and meant every project wrote the same counter with
its own bugs. The count now belongs to the configuration and the engine measures
it: see [Expectations](#expectations).

**Directives arrive next to the existing tests, not instead of them.** In the
pilot the existing test file was kept untouched and the directives were added
alongside it. Nothing is migrated until its x3 equivalent has been seen to go
red on a deliberately broken input.

## Gaps we know about

Stated plainly, because a capabilities document that lists only strengths is a
sales page.

- **A baseline is coarser than the finding it holds.** `comments` identifies a
  finding by rule and file, so a file that already owes one over-long block can
  grow a second one without the gate seeing it; the debt is cleared per file,
  which is also how it is paid. Two identical findings in one file collapse into
  one entry everywhere for the same reason: an identity that counts occurrences
  would move again the moment one of them was fixed.
- **An exclusion is only as narrow as somebody wrote it.** `ignore` entries are
  regular expressions over the matched value, and nothing checks that one of
  them is not swallowing a real credential — only that it swallows *something*.
  The dead-exclusion red catches the rule nobody needed; it cannot catch the
  rule that was written too wide.
- **Exclusions apply to the scan, not to redaction.** The same patterns decide
  what a recorded ledger masks, and that path deliberately ignores `ignore`: a
  value nobody has to hide is cheap to mask, and a value that should have been
  hidden is not.
- **`of: files` counts one directory, not a subtree.** A directory that splits
  its files into new subdirectories shrinks by that measure even though the same
  files are still there; the count says how crowded one folder is, which is the
  question it was written for.
- **Nothing checks that a baseline was reviewed.** `-update-baseline` refuses
  growth, but the first write accepts whatever the tree owes that day. The file
  is in the repository and shows up in a diff; that review is the only control
  there is.
- **Expectations count directives, and only from `scan`.** They say a minimum,
  never a maximum, and they cannot say "these two exact directives" - a file
  carrying three guards satisfies a `min: 2` that was written for two other
  ones. The other checkers report findings rather than directives and have
  their own `empty_scope` protection instead.
- **`update` verifies a checksum, not a signature.** It proves the file it
  installed is the file somebody expected; it cannot prove who wrote the
  release. Without `update.pin` that expectation comes from `SHA256SUMS.txt`,
  which ships inside the release it describes — a source that can rewrite the
  binary can rewrite the list beside it. `update.pin` moves the expectation into
  the project's own repository and closes that particular hole, but it opens no
  identity: it binds *bytes*, and it is worth exactly as much as the review of
  the commit that introduced the line. Nothing here checks a key.
- **A pin has to be maintained by hand.** No command writes or refreshes one,
  and the engine cannot tell a deliberate upgrade from a mistake, so every new
  release is refused until somebody writes its checksum down. That friction is
  the feature; it is still friction, and a project that pins must budget for it.
- **The minimum version gate is enforced from v0.30.0 on.** An older binary
  reads the same configuration file and never sees the requirement, so pinning
  a minimum protects you from binaries newer than the gate itself, not from
  every old one.
- **The cache is per file, not per project.** A checker whose answer depends on
  more than one file at a time — `arch`, `freeze`, `docs`, `boxes` — does not
  use it, and a change in one file still costs a full pass for those. `lang`
  can be slower with the cache than without it on a project where it finds
  thousands of findings; the Speed section gives the numbers.
- **Parallel replay is opt-in and all-or-nothing.** `workers` applies to the
  whole ledger; there is no way to say that these five interactions are
  independent and those two are not. A suite with any ordering constraint runs
  sequentially or declares `carry` and is refused parallelism.
- **A masked value cannot be sent back.** Recording redacts credentials, so
  replaying an authenticated suite sends the requests without them. The count
  is printed, but the run behind an authorisation wall is not the run that was
  recorded.
- **Outbound recording is plain HTTP only.** `x3 outbound` refuses `CONNECT`
  rather than terminating TLS, so a service reached over HTTPS cannot be
  recorded yet. Matching is on method, host and path: two calls to the same
  path with different bodies are told apart only by their order.
- **A recording is as good as the traffic it saw.** Nothing measures coverage: a
  ledger of two requests looks exactly as green as a ledger of two hundred.
- **The language gate speaks one language.** `en` is the only embedded
  dictionary, so `allowed` accepts nothing else today. A dictionary is also a
  blunt instrument: an English word the list does not have (a rare technical
  term) is red until it is allow-listed, and a foreign word that happens to be
  an English word (`kilim`, `sultan`) passes. The non-ASCII rule is what catches
  most of the second case.
- **A lane is paths, not intent.** `scope` can see that a commit touched two
  places; it cannot see whether the second one had to move with the first.
  That judgement is the reasoned crossing, and the gate only insists the
  reason be written down.
- **`syntax` carries one parser.** JSON, and nothing else. Everything beyond it
  is an external command the project installs and names, which means a machine
  without that command cannot run that check - it says so rather than passing.
- **A long block is a proxy, not a judgement.** The gate counts lines; it cannot
  tell a necessary table from a paragraph nobody needed. That is what the
  reasoned exemption is for, and why the ratio only warns.
- **`containment` reads paths, not contents.** A part is recognised by a key at
  the start of a path segment, so a table name inside a file, or a part whose
  name nobody prefixed, is invisible to it. `deps` with `match: "literal"` is
  the kind that reads names inside files.
- **`consistency` reads three shapes and no more.** Typed string constants, JSON
  keys, and one regular-expression capture. A set that lives anywhere else — a
  database table, a generated file, a YAML document — cannot be compared yet.
- **`vocabulary` reads words, not meaning.** A term that is also an ordinary
  word turns every innocent use red, and a term under three letters is never
  read at all.
- **`duplication` compares text, not meaning.** Two bodies that differ by one
  renamed variable are two different bodies to it. That is deliberate — the
  alternative catches deliberately separate code and turns the gate into noise —
  but it means a copy is easy to hide.
- **`exposure` sees tags, not serialization.** An untagged field written out by
  a marshaller is invisible to it, and a tagged field in a type nobody
  serializes is still read.
- **`flow` does not follow a copy.** Once the value is assigned to another name,
  what happens to that name is invisible — the assignment itself is the finding.
  Following it would need type resolution over the whole program. The `import` and `symbol` matchers read Go only,
  so pointing their `sources` at a text glob leaves them with nothing to read.
- **`symbol` reads names, not types.** It sees `client.Retry` without knowing
  what `client` is, so a forbidden call reached through an interface, a function
  value or a wrapper is invisible to it, and two different types with the same
  method name are the same name to it. Type resolution would need the whole
  program; this reads one file at a time.
- **`pairing` reads names, not meaning.** `references-a-declaration` is a text
  search for a declared name, so a counterpart that merely mentions the name in
  a comment counts, and a short name that occurs by accident counts too.
- **A text file cannot carry an exemption.** `//x3:allow:arch:` is a Go comment.
  A `literal` finding in SQL or JSON is answered by fixing it, by `alsoAllow`,
  or by narrowing `sources` — not by silencing that one line.
- **The inward form does not see a component's inside.** `to` + `allowFrom`
  answers "who reaches in from outside", so one checker importing another
  checker inside the same component passes. Splitting them into instances needs
  a single star in the pattern, which a list of fixed directories cannot have;
  until the pattern language grows, that rule is written as one `deny` rule per
  component or not at all.
- **`boxes` measures evidence, not completion.** A criterion is a file or a
  pattern, so a box whose criteria are weak passes while the work is half done.
  What the gate guarantees is that the list and the repository agree, not that
  the criteria were well chosen.
- **`secrets` reads formats, not meaning.** A credential with no recognisable
  shape - a long random password in a variable - is invisible to it, and a
  string that happens to match a shape is red even when it is an example. The
  exemption exists for the second case; nothing covers the first.
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

This repository holds itself to the rule it ships: a change under `internal/` or
`cmd/` must carry a change under `docs/` in the same diff. The gate is
[`x3 docs`](#x3-docs) reading the `docs` section of this repository's own
`x3.json` — the same command any project would run.

A reasoned skip is written in the commit body:

```
docs: none - <why the reader loses nothing>
```

For the run before the commit exists, pass the same line with `-reason`. The
marker with nothing after it is red, on purpose.

---

<!-- x3-dist version=v0.36.1 capabilities=eb404947df01eecd302c7caeb20ab3e36c0e611fc089a9031c6c6f539c7c8a3f template=27dd89792d6c3fadeaa61f5d04ffd541f54e90e6867c6063b9ccc48325519be2 -->
