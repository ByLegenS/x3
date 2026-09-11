# Directives in the source

[The pages](INDEX.md) - [what x3 is](../README.md)

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

See **scan exit codes** in [REFERENCE.md](../REFERENCE.md#scan-exit-codes).

Exit codes are the same for every command. A tree with **no directives at all**
exits `0`, so an exit code alone cannot tell "everything passed" from "nothing
was checked" — read the counts too, see
[Using x3 from another project](releases.md#using-x3-from-another-project).

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

See **the directive dictionary** in [REFERENCE.md](../REFERENCE.md#the-directive-dictionary).

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
`rule`'s code path — see [Gaps we know about](gaps.md#gaps-we-know-about).

### `//x3:case: <payload>`

**Catches:** an inline example — one input and its expected output — next to the
function instead of in a test file. **`decl` scope only**: an example belongs to
one declaration.

The payload's shape is
`[given=(<statements>) ]in=(<args>) [out=<want> ][then=(<propositions>)]`. The
argument list may be empty, **one of `out=` and `then=` is required** and both
may be written,
and the closing `)` is found by **counting** rather than by taking the last one
on the line, so nested calls fit on both sides: `in=(f(1), 2) out=ErrX`.

```go
//x3:case: in=(1) out=1
func (w *Wallet) Add(n int64) int64 {
```

`x3 scan` reads the shape, not the values; [`x3 case`](case.md#x3-case) calls the
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

See **scan error codes** in [REFERENCE.md](../REFERENCE.md#scan-error-codes).

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

See **the scan report fields** in [REFERENCE.md](../REFERENCE.md#the-scan-report-fields).

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

See **expectation fields** in [REFERENCE.md](../REFERENCE.md#expectation-fields).

```
BLOCK expectation_not_met: the ledger package keeps its guards
	0 verified guard directive(s), the configuration requires 2
```

**Only verified directives count.** A directive the scan marked red counts as
zero — otherwise emptying a `//x3:guard` would satisfy the expectation that
exists to notice its removal. **A stale `paths` is red, not silent**: matching
nothing gives zero, and zero meets no expectation. The report is read, never
written, so identical sources still produce identical bytes.

**An expectation brings its subject into scope.** The scan does not walk into
`testdata`, `vendor` or `node_modules`, so a count about a fixture tree could
never be met from the root — and its red would read *"the directives were
deleted"* when the truth is *"the run never looked there"*. A rule's `paths`
therefore **open what they name**: the walk enters a skipped directory only for
the files a rule declares, and reads nothing else in it.

**Paths are written against the configuration and read against the run.** A run
rooted at a subdirectory judges only the expectations reaching into it, with
their paths taken relative to that root; one about somewhere else is not asked.
The law does not slacken: a run rooted at the configuration's own directory
judges **every** expectation, and each has already pulled its subject into
scope. A run rooted **outside** that tree is an error, not a pass.

<!-- x3-dist version=v0.88.0 capabilities=705c7f5ac735e349fe03fb1384c40591cad27fbc50c96ba3a2ae0545428c6440 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
