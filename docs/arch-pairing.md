# Does anybody touch this file

[The pages](INDEX.md) - [what x3 is](../README.md)

## `pairing` — does anybody touch this file?

```json
{ "kind": "pairing", "sources": ["internal/**/*.go"],
  "counterpart": "sibling:*_test.go", "requires": "references-a-declaration" }
```

**Not coverage** — the question is whether any file at all names this one. There
are two counterpart forms and they are **not the same question**.

| Form | What it looks for | `*` means |
|---|---|---|
| `sibling:<template>` | the subject's **namesake**: `beta.go` asks for `beta_test.go` | the subject's own base name |
| `in-directory:<pattern>` | **any** neighbour matching the pattern | an ordinary glob star |

Read as a glob either string also says which files are counterparts, and those
are never subjects. `requires` is `exists` (default) or
`references-a-declaration`; a subject with no declarations cannot be asked the
second question.

The directory form exists because one suite file can cover a whole directory.
Measured on a fixture where `suite_test.go` names both `service.go` and
`ledger.go`: `sibling:` reports **2 violations** — it wants `service_test.go`
and `ledger_test.go`, and both would be empty files — while `in-directory:`
reports **none**. Under `references-a-declaration` **one** neighbour naming one
declaration is enough; asking every match to name it would turn "does anybody
touch this file" into "does everybody".

It is not an off switch. In the same fixture a declaration no neighbour names is
red (`N file(s) here match "*_test.go" and none names anything declared here`),
and a directory with no matching file at all is red too (`no file in this
directory matches "*_test.go"`).

### What counts as a subject

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

<!-- x3-dist version=v0.128.0 capabilities=796b04d7c3d74701288af9fa0577abc2b3913672a31190f52dd3e2d10c7a8e1e template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
