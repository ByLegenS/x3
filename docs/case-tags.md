# Examples behind a build tag

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, examples behind a build tag

Some of what an example needs is not hidden behind an import but behind a
**build constraint**. A package's database helpers commonly sit in a file that
opens with

```go
//go:build integration
```

and that file is simply **not part of the package** unless the build carries
the tag. An example whose setup calls such a helper does not fail its
assertion; it never compiles. The run says `does_not_build — undefined:
fixtureRate`, and no amount of declaring imports helps, because the name is not
in another package — it is in *this* package, in a file the compiler was not
given.

### The declaration, and the run that carries it

A file says which tags its examples need:

```go
//x3:tags: integration

package usage
```

and a run says which tags it carries:

```
x3 case -tags integration ./...
```

or, for a project whose default run always carries them:

```json
{ "case": { "tags": ["integration"] } }
```

The flag **replaces** the setting rather than adding to it: someone narrowing a
run by hand would otherwise measure a tree the setting silently widened. A
declaration names tags and only tags — `!live`, `a && b` and the rest of the
constraint algebra stay in Go, where they already work; the only question asked
here is whether the run carries this name.

The directive binds the **file**, exactly as `//x3:import:` does, and is written
above the `package` clause for the same reason: a declaration sitting over one
declaration reads as if it bound only that one, and anywhere else it is
`malformed`. What the tag reaches, though, is wider than the file — in Go a
build tag decides which files the whole **package** is assembled from. The file
declares because the file is what needs it; the report says which tags the run
carried, so the wider reach is never a guess.

### An example a run does not run, said out loud

When a run does not carry a tag a file declares, that file's examples are
**deferred**: they do not run, and the run does not pretend they did.

```
DEFERRED rate.go:14 (Scale): the run does not carry build tag(s) integration
x3 case: 2 example(s) in 1 package(s) - 1 passed, 1 deferred, 0 finding(s)
```

Every deferral is printed with its file, its line and the declaration it calls,
and the report carries the same list under `deferred[]` with a `deferred` count
in the summary. A gate that wants to refuse any unrun example can read one
number and say so.

This is not a skip, and the difference is the whole point. **A skipped example
is green without having run**, which is the one thing an example must never be.
A deferral is decided **before anything is compiled**, from a declaration
written in the source, by a switch the run itself sets — so the run can name
every example it is not running, ahead of running the rest. An example that
starts and then bails out remains what it always was: `never_ran`, and red.

Deferral also hides nothing. The same example, once the tag is carried, runs and
is judged: a wrong expectation is `example_failed` exactly as it would be
anywhere else. And an example that declares nothing is never touched by any of
this — it runs in a tagged run and in an untagged one alike.

### Tags that do nothing are red

A tag on a command line is a claim that it does something. Two reds keep the
claim honest, and they are the same law read from either end:

- the run carries a tag **no example asks for** — `dead_tag`, named on the
  configuration file or on `-tags`, whichever declared it;
- a file declares tags and **has no examples** — `dead_tag` on that line: the
  declaration binds that file's examples, and there are none to bind.

Without the first, a gate script could keep `-tags integration` long after the
last example that needed it was renamed, and go on reporting green over a
narrower tree than anyone believes it measures. The second catches the same
mistake one file earlier.

### An example that never ran can be refused

A deferral is loud and not red, which is right for a run narrowed on purpose
and wrong for a gate: drop one entry from `case.tags` and every example behind
that tag defers, the run says so, and the exit code is still `0` — the hole a
skipped test leaves, one level up.

```json
{ "case": { "deferred": { "policy": "block", "max": 0 } } }
```

| Field | What it is |
|---|---|
| `policy` | `warn` (default) or `block` |
| `max` | in `block`, how many deferrals are allowed; 0 when not written |
| `listed` | how many examples the finding names; 5 when not written |

The fields and their defaults are the ones `test.skips` carries, because two
places asking one question should take one answer under one name. The default
stays a warning — **a declaration cannot redden an existing tree by itself** —
and the finding is charged on the configuration, naming the examples. It is
asked on every run, narrowed or not: it measures this run rather than judging
a declaration, so the scope rule above does not apply to it.

The run-side question is asked of a tag from the configuration **only when the
configuration file lies inside the tree being run** — the same rule that governs
`dead_import`, for the same reason: a repository-wide setting judged from a
single directory would call a correct declaration dead. A tag given on the
command line is not held back that way; it was typed for this run, over this
scope, and "nothing here asks for it" is both true and worth hearing.

<!-- x3-dist version=v0.148.0 capabilities=55e7b1ecf9f883aca1c04bd910648430f82c11bba1b1e2db63348f9a623d62d2 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
