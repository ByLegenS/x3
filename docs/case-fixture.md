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
variables, constants.

### The tag a fixture may share

The constraint carries that tag alone, or beside others joined by `||`:

```go
//go:build x3fixture || live
```

`x3fixture` is never passed to any build, so the product still ignores the file;
a run carrying `live` sees it. A package whose tagged tests need the same setup
writes it once instead of twice, and the tagged build compiles against it:

```
$ go build ./shared            # clean; store_fixture.go is ignored
$ go vet -tags live ./shared   # clean; the live test calls warmStore
```

Inside `&&`, or after `!`, the file joins neither build and the run says so with
`malformed`: the first asks for two tags where one is never given, the second
asks for the tag's **absence**.

### What a fixture may not hide

Five findings, each with a measured control arm:

| Written in a fixture | Code | Why |
|---|---|---|
| a declaration no example and no other fixture line names | `dead_fixture` | the file is outside the product build, so neither the compiler nor the dead-code gate can see it die |
| a declaration a plain `_test.go` still names | `fixture_used_by_a_test` | that file builds without `x3fixture`, so the declaration is not there and the build fails on an undefined name |
| a declaration a tagged `_test.go` names by **declaring it again** | `fixture_redeclared_by_a_test` | that build carries its own copy and leans on nothing; a tag union joining the two would break it on a duplicate |
| `func TestBumpCounts(t *testing.T)` | `test_in_a_fixture` | no runner reaches the file, and no count of test files would ever see it |
| `//x3:case:` | `case_in_a_fixture` | an example belongs on the declaration it proves, and this file is built *for* the examples |

`fixture_used_by_a_test` is the question a migration asks **before** it moves a
carrier: the engine reads the package's test files the way it reads its
examples, and names the declaration a test still leans on. A test file that
demands a tag the fixture carries beside `x3fixture` is not asked — the two are
in one build. Neither is a method: a method is reached through a selector, and
the right side of a selector is the one name this reading does not count.

The reading stops at the package's **own** test files. A file in the external
test package (`package p_test`) compiles apart from the fixture whatever the
tags say, so a name it writes is never an undefined name in the fixture's
build; it is that package's own name, and it is not asked about — in the dead
reading either, where counting it would keep a dead declaration looking alive.
Measured on a pilot: six findings on one fixture, every one of them from
`package p_test` files that carried their own copies, while `go vet` was clean
under the tags they demanded.

The two answers are separate codes because they ask for opposite moves: a test
that **leans** on the fixture asks for the tag union, and one that **declares
the name itself** asks for the copy to go.

```
$ x3 case ./fixture-outside     # package p_test names warmStore, declares its own
x3 case: 1 example(s) in 1 package(s) - 1 passed, 1 fixture file(s), 0 finding(s)

$ x3 case ./fixture-redeclares  # //go:build live test file declares its own copy
store_fixture.go:9: fixture_redeclared_by_a_test
	fixture declaration warmStore is declared again by store_live_test.go, which
	builds without "x3fixture" and carries its own copy; that build does not lean
	on the fixture, and a tag union joining the two would break it on a duplicate
```

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
| `//go:build x3fixture && linux`, and `//go:build !x3fixture` | `malformed` |
| `go build` / `go vet` on the green package | clean; the fixture is in `IgnoredGoFiles` |
| plain `_test.go` names the helper the fixture declares | `fixture_used_by_a_test`; `go vet` on that package: `undefined: warmStore` |
| fixture says `x3fixture \|\| live`, a `//go:build live` test names the helper | no finding; `go build` clean, `go vet -tags live` clean |
| `package p_test` (external) declares and names the same helper | no finding; that package never sees the fixture |
| a `//go:build live` test declares its **own** copy of the helper | `fixture_redeclared_by_a_test` |

<!-- x3-dist version=v0.289.0 capabilities=e2b1c902dafbfc124d29f232a1f3e1807c6357f33259deee7c9a3811eaf626d6 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
