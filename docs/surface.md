# The exported API, which may only grow

[The pages](INDEX.md) - [what x3 is](../README.md)

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

<!-- x3-dist version=v0.73.0 capabilities=02e34c7d4650e29421d327be97df8b9ff842a32f83d6a8cc046718f71f896143 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
