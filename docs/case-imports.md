# The names an example may reach

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, the names an example may reach

The page before this one is the run: the directive, the state given to it, and
the aspects of a result an example asserts. This one is the vocabulary an
example may reach for beyond its own file — and where that vocabulary is
declared.

### A name the source file cannot import

An example of an HTTP handler needs `net/http/httptest`, which no source file
imports: **an unused import is a compile error in Go**. Such a name is not one
the author forgot but one the file *cannot* carry — as with a project's own test
helpers — so it is **declared**, and only a package an example names is written
into the generated test.

```go
//x3:case: given=(rec := httptest.NewRecorder(); req := httptest.NewRequest("GET", "/hi?name=ada", nil)) in=(&Desk{Greeting: "hello"}, rec, req) then=(rec.Body.String() == "hello ada")
func (d *Desk) Greet(w http.ResponseWriter, r *http.Request) {
```

**The file's own imports win**, because the example is read against that file. A
declared import makes a name resolvable, never a value correct.

**A declaration no example names is red** (`dead_import`), including one the file
already provides: a declaration is an escape hatch, and an entry that serves
nothing becomes the dead line that hides a name someone will really need.

### A name the path cannot spell

A package's name is not required to be the last segment of its path. A directory
`lib/text/core` may hold `package typeset`, and then no example can name it at
all: the engine derives the qualifier from the **path**, and the path says
`core`. Adding the path to a pool does not help, because the pool is read
the same way.

So a declaration may carry the name, exactly as Go's own import syntax does:

```json
{ "case": { "imports": ["net/http/httptest", "typeset \"example.com/lib/text/core\""] } }
```

The name is taken at its word — the engine writes the import into the generated
test **with that alias**, so the binding holds whatever the package calls
itself. A name that is not a plain identifier, or is `_` or `.` (neither
produces a qualifier an example could write), is a configuration error, not a
finding. One name may not be declared twice: an example's qualifier has to say
which path it went to.

### Where a declaration may be written

The same declaration may live in either of two places, and they are read
**narrow first**: the file's own imports, then the declarations written in that
file, then the pool in the configuration.

```go
//x3:import: typeset "example.com/lib/text/core"

package caption
```

The directive binds the **file** — an example sees the names its own file sees —
so it is written above the `package` clause; anywhere else it is `malformed`,
because a declaration sitting over one declaration reads as if it bound only
that one.

**Why both.** A pool in the configuration is right for a name a whole repository
needs (`httptest` in forty files) and wrong for a name one file needs, for a
reason that is not about taste: the declaration and the example that takes it
**must arrive together**. Declared alone, it is a dead declaration and red;
written alone, the example does not compile and is red. Where the configuration
and the sources are edited by different hands — a migration in which one person
owns `x3.json` and another owns the packages — those two reds are a deadlock,
and neither half can be landed first. A declaration that lives in the same file
as the example that needs it has no such seam.

The law does not loosen anywhere: a declaration in a file that no example in
**that file** names is `dead_import`, reported on its own line. It is asked only
of a file whose examples could answer — a file with examples the parser could
not read measures nothing about its declarations, and a second red charged on
top of the real one names the wrong culprit. A file carrying declarations and no
examples at all is not that case: there, nothing could ever have taken them.

**But that question is only asked where the declaration itself is in scope.** The
pool is declared for a whole repository while a run can be narrowed to a single
directory, and the example that takes an entry is usually *outside* the
narrowing — there, "no example names it" is simply untrue. Nor is it demoted to a
warning: softening a false sentence does not make it true, and an author who
watches a correct package go red will go and break the example that works. So
the measure is the scope itself. The file a `dead_import` blames is the
configuration, and the declaration is judged when that file lies inside the tree
being run — which a whole-tree run always does, so the law does not loosen. What
stops is a narrow run answering a question nobody asked it.

<!-- x3-dist version=v0.88.0 capabilities=705c7f5ac735e349fe03fb1384c40591cad27fbc50c96ba3a2ae0545428c6440 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
