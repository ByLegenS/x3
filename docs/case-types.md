# The type an example declares

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, the type an example declares

The page before this one is about names an example reaches for. This one is
about a name that does not exist anywhere yet: the **fake** an example needs
and the tree has no reason to carry.

### What a setup cannot write

An example gives its state with `given=`, and a setup can write any value. It
cannot write a **type**, because the one thing a type needs in order to stand
for something is methods, and *Go does not allow a method inside a function
body*. Three measured answers, all from the same example of a function that
takes an interface:

| Written | What the run says |
|---|---|
| nothing declared | `does_not_build`: `undefined: fakeSink` |
| `given=(type fakeSink struct{}; ...)` | `does_not_build`: `fakeSink does not implement Sink (missing method Take)` |
| a method inside `given=` | `does_not_build`: `missing ',' in argument list` |

Adding a fake to the tree answers it, and costs the tree a type that exists for
no other reason. A declaration answers it without that cost:

```go
//x3:type: type fakeSink struct{ got int }
//x3:type: func (f *fakeSink) Take(n int) error { f.got += n; return nil }

package drain

//x3:case: given=(s := &fakeSink{}) in=(s, []int{1, 2}) out=nil then=(s.got == 3)
func Total(s Sink, parts []int) error {
```

The declared text is written into the **generated test**, at package level,
exactly as it stands — and only if an example names it. The tree being measured
is not touched: the same overlay that carries the generated test carries the
type, and nothing is left behind when the run ends. What the **declaration**
names is read too, not only what the examples name: a fake's method signature
reaches for packages (`context` here) that no example writes, and an import
chosen from the examples alone would leave the declaration uncompilable.

One declaration per line, and the line binds the **file** — it is written above
the `package` clause, like `//x3:import:` and for the same reason: a
declaration sitting over one declaration reads as if it bound only that one.

### Only a type, and the methods that make it one

A declaration carries a type declaration, or a method whose receiver is a type
**the same file declares**. Anything else — a variable, a constant, a free
function — is `malformed`, and the message says where that belongs: a value an
example needs is written with `given=`.

The receiver law is the sharper half. A method on a type the *package* already
has would compile only inside the generated test, and the type would then be
one thing while the gate runs and another thing when the code ships:

```go
//x3:type: func (m Meter) Secret() int { return 7 }
```

> `the receiver type Meter is not declared in this file; a declaration may not
> add a method to a type the package already has`

### A declaration no example names is red

`dead_type`, reported on its own line — the same law as `dead_import` and for
the same reason: a declaration is an escape hatch, and an entry that serves
nothing becomes the dead line that hides a name someone will really need. It is
asked only of a file whose examples could answer, so a file whose examples the
parser could not read is not charged a second red on top of the first.

**Methods are not asked separately.** A method is the surface of its type, and a
method written to satisfy an interface is needed precisely *without* being
called by any example. A law that called the uncalled ones dead would turn the
fake's reason for existing into a finding.

### When the declaration itself does not compile

A declared name can collide with one the package already has, and then the
compiler speaks on the declaration's line — which belongs to no example. Blame
by line number would find nobody and charge the whole package, so the engine
records **which example named each declared type** and the finding lands there:

> `Meter redeclared in this block; the type was declared because this example
> names it`

The neighbours in that package still run, and still pass.

### What is not here

There is no repository-wide pool of types, as there is for imports: a pool of
fakes would be a second source tree written in comments, visible to the compiler
and invisible to the reader. A fake several packages really share is a package,
and an example reaches it with `//x3:import:`.

<!-- x3-dist version=v0.105.0 capabilities=c84b4eb7f15d69ec3de7d110114238de1bf6e9273e7f333acd5a8f192af87562 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
