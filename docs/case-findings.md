# What an example may name, and what a run says

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, what it may name and what it says

The page before this one is the run: the directive, the state given to it, and
the aspects of a result an example asserts. This one is the vocabulary — the
names an example may reach for beyond its own file, the codes a run answers
with, and what it writes down.

### A name the source file cannot import

An example of an HTTP handler needs `net/http/httptest`, which no source file
imports: **an unused import is a compile error in Go**. Such a name is not one
the author forgot but one the file *cannot* carry — as with a project's own test
helpers — so it is declared in the setting (`case.imports`, below) as a **pool
of candidates**: only a package an example names is written into the test.

```go
//x3:case: given=(rec := httptest.NewRecorder(); req := httptest.NewRequest("GET", "/hi?name=ada", nil)) in=(&Desk{Greeting: "hello"}, rec, req) then=(rec.Body.String() == "hello ada")
func (d *Desk) Greet(w http.ResponseWriter, r *http.Request) {
```

**The file's own imports win**, because the example is read against that file. A
declared import makes a name resolvable, never a value correct.

**A declaration no example names is red** (`dead_import`), including one the file
already provides: a pool is an escape hatch, and an entry that serves nothing
becomes the dead line that hides a name someone will really need.

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

### Findings

See **case finding codes** in [REFERENCE.md](../REFERENCE.md#case-finding-codes).

The first three are answers the toolchain gave; the last four are refusals made
**before** anything runs.

### Settings

Optional — an example lives in the source, not the configuration:

```json
{ "case": { "exclude": ["internal/legacy/**"], "imports": ["net/http/httptest"], "timeout": "2m" } }
```

`imports` is the pool above. `timeout` (default `1m`) is applied to the test binary **and** to the toolchain
call around it; only the first would leave a run that hangs downloading a
dependency waiting forever.

### The report

```json
{ "version": 1, "root": ".", "config": "x3.json",
  "findings": [ { "file": "wallet.go", "line": 8, "target": "Add",
                  "code": "example_failed", "message": "out[0] = 5, want 6" } ],
  "summary": { "files": 1, "packages": 1, "cases": 1, "passed": 0, "findings": 1 } }
```

`passed` is counted separately from `findings` on purpose: "no findings" and "no
examples" are not the same sentence.

<!-- x3-dist version=v0.70.0 capabilities=d6bca49b1ee20fb16cf56855193fb72748bc6213792c4f4e81682cf9ef31d4b0 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
