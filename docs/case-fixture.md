# The setup every example in a package shares

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, the setup every example in a package shares

The page before this one is about a fake an example declares in one line. This
one is about the other half of the same problem: a setup **twenty examples
share**, too long for a line and too costly to copy.

### The wall a migration hits

A test file holds helpers, and examples may call them — but only while that
file exists. Measured on a repository migrating its tests into examples: of the
test files still standing, most were no longer tests at all. Half of them
declared **no `Test` function**; what they still carried were helpers that the
`given=` of an inline example called. The file had become a carrier, and the
last stop of the migration was another `_test.go`.

`//x3:type:` does not answer it. It is one line, deliberately: a variable, a
constant, a free function written there is `malformed`. Copying the setup into
each `given=` answers it by writing the same lines N times, none of them
measured.

### A file that declares itself

A **fixture** is an ordinary Go file carrying one build constraint:

```go
//go:build x3fixture

package green

import "testing"

// warmStore is the setup the examples of this package share.
func warmStore(t *testing.T) *Store {
	t.Helper()
	return &Store{hits: seed}
}

const seed = 3
```

The constraint is not decoration; it is the whole mechanism. Measured on that
package:

```
$ go list -f '{{.GoFiles}} ignored={{.IgnoredGoFiles}}' ./green
[store.go] ignored=[store_fixture.go]
```

`go build` and `go vet` never read the file, so a fixture cannot leak a byte
into the product. `x3 case` reads it and writes it into the same overlay that
carries the generated tests, under the package's own name — so the file joins
the **test** build, may import `testing`, and the helpers it declares are in
scope for every example in that package:

```go
//x3:case: given=(s := warmStore(t)) in=(s) out=4
func Bump(s *Store) int {
```

The file is **declared, not discovered**: the engine knows it by the constraint
it reads, and no project writes a list of fixture paths anywhere. Scope is the
package, and what may be declared is unrestricted — types, methods, functions,
variables, constants. The constraint carries that tag **alone**; inside a
compound constraint (`x3fixture && linux`) the file would join neither build,
and the run says so with `malformed`.

### What a fixture may not hide

Three findings, each with a measured control arm:

| Written in a fixture | Code | Why |
|---|---|---|
| a declaration no example and no other fixture line names | `dead_fixture` | the file is outside the product build, so neither the compiler nor the dead-code gate can see it die |
| `func TestBumpCounts(t *testing.T)` | `test_in_a_fixture` | no runner reaches the file, and no count of test files would ever see it |
| `//x3:case:` | `case_in_a_fixture` | an example belongs on the declaration it proves, and this file is built *for* the examples |

`dead_fixture` does not ask about methods or `init`, for the same reason
`dead_type` does not: a method is the surface of its type and is needed
precisely without being called. It is not asked of a package whose examples the
parser could not read — a red written for something unmeasured lands on top of
the real red.

### The count is spoken

Every run says how many fixture files it read:

```
x3 case: 2 example(s) in 1 package(s) - 2 passed, 1 fixture file(s), 0 finding(s)
```

The line is there because no other gate can print it. A fixture is not named
`_test.go`, so every count of standing test files walks past it; a file kind
nobody counts is a file kind that grows, and a melting test suite would read as
finished while its setup quietly moved house.

### Control arms

| Arm | Result |
|---|---|
| example calls a helper the fixture declares | passes; `1 fixture file(s)` |
| same package, fixture deleted | `does_not_build`: `undefined: warmStore` — and the example that does *not* name it still passes |
| fixture declares a name nothing reaches | `dead_fixture` on that line; the example still passes |
| `func TestBumpCounts` in the fixture | `test_in_a_fixture` |
| `//x3:case:` in the fixture | `case_in_a_fixture` |
| `//go:build x3fixture && linux` | `malformed` |
| `go build` / `go vet` on the green package | clean; the fixture is in `IgnoredGoFiles` |

<!-- x3-dist version=v0.283.0 capabilities=5aa4b22a2ef684006ecfe675aeba4575628ec4e1904f5aebab18307bfbf751f2 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
