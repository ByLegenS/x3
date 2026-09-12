# Which files the rule is about

[The pages](INDEX.md) - [what x3 is](../README.md)

## `pairing` — which files is the rule about?


```json
{ "kind": "pairing", "sources": ["**/*.go"], "counterpart": "in-directory:*_test.go",
  "requires": "references-a-declaration", "subjects": "files-that-declare-a-function" }
```

A file that declares an interface, a struct and nothing else has no behavior a
test could touch. Asked for a counterpart anyway it produces a debt that cannot
be paid — the only way to close it is an empty test file — and a debt list made
of those is a list nobody reads.

`subjects` says what the rule is **about**. The two readings are

| Value | A subject is |
|---|---|
| `every-file` (default) | every file `sources` matches |
| `files-that-declare-a-function` | only a file that declares something that **runs** |

**Not an exemption, and not an exclusion either.** An exemption overrides a
measurement, so it needs a reason. `exclude` names a **path** and is therefore a
claim about the tree — this one exists — which is why a dead exclusion is red.
`subjects` names a **property** and claims nothing about the tree, exactly as
`sources: ["**/*.go"]` is a true statement of scope in a tree where every file
is Go. So there is no dead-declaration law here to run.

**What runs.** A function declaration — `func f()`, a method (a receiver does
not make a body less of a body) and `init` (its name cannot be called, but it
runs) — **or** a function literal inside a declaration, `var Hook = func() { … }`.
A function *type* (`type Handler func()`) is a signature, and a name bound to
somebody else's function (`var now = time.Now`) is a label on another file's
body; neither is behavior here. Build constraints are not evaluated: the file
is read as source, so both halves of a `//go:build` pair are judged the same
way. A file the engine does not parse as Go declares no function at all, so a
source set holding non-Go files loses **all** of them to this reading.

**The elimination is counted.** `rules[].eliminated` in the report, a line of
its own on stderr, and if it takes the last subject away the rule is red with
`empty_scope` naming what emptied it — an elimination nobody can see is the
quietest way to turn a gate off.

<!-- x3-dist version=v0.155.0 capabilities=bd64cc3512a9fc65db0936bc54546917afbce4d7828169eb8afe0fe2307d36ce template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
