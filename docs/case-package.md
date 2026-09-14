# An example on the package itself

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, an example on the package itself

**What it catches:** a package whose whole job is a **registration side effect**
— importing it *does* something — making a claim nothing runs, because there is
no declaration to write an example above.

Such a package exists in most trees. It carries blank imports and nothing else:

```go
// Package engines imports every voice engine, from one place.
package engines

import (
	_ "app/internal/ai/voice/chain"
	_ "app/internal/ai/voice/hybrid"
)
```

`grep -cE '^func ' engines.go` answers `0`. The claim the file makes —
*importing this package registers those engines* — is real and measurable, and
it had nowhere to be written: an example goes above a **declaration**, and here
there is none.

### The payload, with no call in it

It is written **above the `package` clause** — the place a file's other
declarations already use (`//x3:import:`, `//x3:tags:`, `//x3:type:`):

```go
//x3:import: voice "app/internal/ai/voice"
//x3:case: given=(e, ok := voice.Lookup("chain")) then=(ok, e != nil)
//x3:case: then=(voice.Count() == 2)
package engines
```

`in=(...)` is **not written**, because there is nothing to call; `out=` is not
written, because nothing returns. What is left is the half that was always doing
the measuring: `given=(...)` sets a state up, `then=(...)` asserts what must
hold. The generated test still compiles **inside the package**, so the blank
imports above it have already run by the time a proposition is read.

### The fence does not move

An example is written above the thing it proves so that it cannot outlive it. A
package example keeps that: it sits in the file that carries the import list,
beside the lines it measures. Delete one of them and the proposition goes red on
its own line. That is measured in both directions — `testdata/loaded` is green,
and its twin `testdata/loaded-red`, one deleted import apart, is red.

### What it refuses

| Written above `package` | Refused because |
|---|---|
| `in=(1) out=1` | a package example has no declaration to call |
| `out=1` | nothing returns, so there is no result for `out=` to compare |
| `given=(n := 1)` | nothing is asserted; `then=(...)` is what it measures |
| `then=(len("ab") == 2)` | it names nothing beyond the language itself |
| `given=(n := 1) then=(n == 1)` | it names nothing beyond its own setup |
| anything, in a `_test.go` file | nothing there would run it |

The last three are one law: an example must be **breakable by the package**. A
declaration example gets that from its call — every run executes a real
declaration. A package example makes no call, so the law is asked of the names:
at least one must be neither created by the example's own setup nor predeclared
by Go. A proposition that can only name itself cannot fail, and a criterion that
cannot fail is not a criterion.

The grammar belongs to the place. A body with no call written **above a
declaration** is still refused, or an example could quietly skip the call it was
written to make.

### What the report says

Package examples are counted apart, inside `cases`, never instead of it. The
engine's own fixture answers:

```json
"cases": 2,
"on_package": 2,
"passed": 2,
```

They stay separate because "this many declarations are covered by an example"
would read too large if a package example were counted among them.

The names an example may reach — a package its own file cannot import — are on
the next page.

<!-- x3-dist version=v0.201.0 capabilities=4d27b4a136bda5010e31cb2176fd3134dddf1c7c105fb91703e147ea84a2f5cf template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
