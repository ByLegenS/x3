# Inline examples that run

[The pages](INDEX.md) - [what x3 is](../README.md)

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

### One aspect of the value, not the whole of it

`out=` compares the **whole** returned value, and not everything a call produces
can be written down. A declaration may return nothing at all and do its work in
the receiver; a returned struct may carry an identity or a timestamp the call
itself generated; and a test may say only *"an error came back"* without saying
which one. For all of those, `then=` asserts an **aspect** of what happened:

```
//x3:case: in=(<arguments>) then=(<propositions>)
```

`then=(...)` holds Go boolean expressions separated by top-level commas, each
evaluated after the call. Two names are bound for them: **`recv`** is the
receiver — the first argument, when the example sits on a method — and
**`out0`, `out1`, …** are the results in `out=`'s own order. Anything `given=`
created is in scope too, and a result nothing measures is not bound to a name at
all, because an unused variable is a compile error in Go.

```go
// The work is the side effect; there is no result to compare.
//x3:case: in=(&Counter{Total: 2}, 3) then=(recv.Total == 5)
func (c *Counter) Bump(n int)

// The call generates the identity, so the whole value cannot be written.
//x3:case: in=("apple") then=(out0.Name == "apple", out0.ID != "")
func New(name string) Record

// "An error came back" — which one is not what the test says.
//x3:case: in=("nope") out=0, _ then=(out1 != nil)
func Parse(s string) (int, error)
```

That last one is why this exists. The rule of the migration is that a
`//x3:case` is derived from the **test**, never from the code. When a test says
only that an error came back and the engine demands the exact value, whoever
writes the example reads the error **out of the implementation** and copies it —
and an example written that way no longer verifies the code, it repeats it.

A proposition must be able to **fail**. One that names nothing the call
returned, nothing the receiver holds and nothing the setup created — `then=(true)`,
`then=(1 == 1)` — compiles, runs, passes and proves nothing, which is the very
state [`never_ran`](#an-example-nothing-ran-is-not-a-green-example) exists to
refuse. Those are `malformed`:

```
add.go:9 (Add): malformed
	the proposition true names nothing the call returned, the receiver, or the
	setup created; it cannot fail
```

The check reads names, so a tautology over a real name (`out0 == out0`) is not
caught by it — see [Gaps we know about](gaps.md#gaps-we-know-about).

When a proposition is false the finding carries its **text**, because an example
may hold several and "then[0] does not hold" would not say which:

```
counter.go:8 (Bump): example_failed
	then[0] recv.Total == 4 does not hold
```

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

### The given state

Some declarations read what nobody passed them: a row in a database, an entry in
a registry, a file on disk. An example for one of those needs a **state**, and
the state has to be there before the call.

```
//x3:case: given=(<statements>) in=(<arguments>) out=<expected>
```

`given=(...)` holds Go **statements**, run in order inside the example's own
subtest, before the call; `in=` and `out=` may name what they bind. The section
is optional, and an example that needs no setup writes none.

```go
//x3:case: given=(c := declare("apple", 40)) in=(c) out=40, nil
//x3:case: given=(c := declare("pear", 7); retire(c)) in=(c) out=0, ErrUnknown
func Price(code string) (int, error) {
```

That the binding is nameable in `out=` is the whole reason the section exists.
Setup written *inside* an argument already worked — `in=(f(t))` compiles — but
its value reaches only that argument, and the pattern this is for hands the
identity back: the setup creates a row, and the call is measured against the
identity of the row it created.

The statements are carried **verbatim and unsplit**. A statement list is not an
expression list: both separators — `;` and `,` — appear *inside* single
statements (`if x := f(); err != nil`, `_, err := f()`), so any rule that cut the
section on one of them would cut valid Go in half and call the remainder
malformed. Unsplit, the only thing that reads the section is the compiler, and a
fault in it is charged to the example's own line like any other.

The engine binds one name: **`t`**, the subtest's `*testing.T`. Setup helpers
take it, so a project's existing ones work unchanged — and because the generated
test is part of the package's *test* build, helpers declared in `_test.go` files
are in scope. Setup is test equipment; this is what keeps it out of production
source.

A setup that **cannot** run is the dangerous case, not one that breaks. A helper
that cannot reach its server calls `t.Skip`, the toolchain exits `0`, and a gate
that only looked for failures would report nothing on every machine without that
server. The rule above covers it: a skipped example is `never_ran` and the run is
red. So three directions are worth measuring separately, and the engine's own
gate measures all three — the setup runs (`0`); the setup is **not applied** and
the same expectation turns red (`1`); the setup is **skipped** and the finding is
`never_ran` (`1`). Without the second, a setup the engine silently dropped would
still look green; without the third, a missing server would.

### A database as the given state

Nothing above knows what a database is. `x3 testdb` hands a command a freshly
cloned database through the environment and `x3 case` inherits it, so the two
compose with nothing third to configure and no directive of their own:

```
x3 testdb run -- x3 case .
```

`x3 case` takes a **directory**, not a package pattern — `./...` is not a path
and a run given one exits `2`.

The setup opens that connection the way the project's own tests already do — from
the variable named in `testdb.dsnEnv` — seeds what the example needs, and hands
back the identity it created:

```go
//x3:case: given=(p := open(t); c := company(t, p, 250)) in=(&Repository{DB: p}, ctx, c) out=250, nil
func (r *Repository) Balance(ctx context.Context, id int64) (int, error) {
```

**The variable is the project's, not the engine's.** `testdb.dsnEnv` names it, so
a repository whose helpers already read `APP_TEST_DSN` keeps them as they are:
neither the helper nor the source has to learn what x3 is. Filling the engine's
default name as well would be worse than useless — a helper reading the wrong
name would then pass too.

**One database per run, not per example**, and that is why no example has to
declare anything. The database is created before the command and dropped after
it whatever the examples do; measured against PostgreSQL 18 over loopback, an
empty one costs **241–350 ms** once, and a tree of 43 examples that ask for no
database ran in **4577/4661/4652 ms** wrapped against **4711/4716/4522 ms** plain
— the same run, inside the noise. An example that never names the connection
never opens one.

**The failure this closes is the skipped setup.** A helper that cannot reach a
server calls `t.Skip`, the toolchain exits `0`, and a suite of database tests
reports nothing on every machine without one — green, for years. Here that
example is `never_ran` and the run is red; the examples beside it that ask for no
database still pass, so the red names exactly what was not measured.

**Bound:** one database serves the whole tree, and packages are run one at a
time, so nothing races — but an example that assumes an *empty* table is
assuming something the run does not promise. Seed what you assert on and assert
on the identity you seeded.

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

What an example may *name* — a package its own file cannot import — and
what a run says when one goes red are on the next page.

<!-- x3-dist version=v0.103.0 capabilities=c085ea2f8775173860b02846284ab6863013fe369140640561fafd940510696a template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
