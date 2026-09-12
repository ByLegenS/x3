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

### What counts as naming it

A counterpart names a declaration when it writes the name as a **token**, not
when the name merely occurs somewhere in its bytes. Naming a fixture after its
subject — `seedFoo`, `newFoo`, `fooHelper` — is ordinary Go, and a substring
reading takes every one of them for a mention of `Foo`: the gate then answers
green, not red, and the debt closes itself. So both edges of the match must be
free of a letter, a digit or an underscore.

| Written in the counterpart | `Foo` is named |
|---|---|
| `Foo()`, `= Foo`, `want(Foo, 1)` | yes |
| `tree.Foo()` — a package qualifier | yes: `.` is not part of an identifier |
| `seedFoo`, `TestFoo`, `FooBar`, `Foo_2` | no |

The blank identifier is not a name: `var _ = os.Stderr` declares nothing a
counterpart could write, and counting it would turn every neighbour green,
because `_` appears in every Go file.

The reading is **lexical, not resolved**: a name inside a string or a comment
counts. Telling those apart needs type checking, and this question looks at a
name, not at a compiler's report.

<!-- x3-dist version=v0.148.0 capabilities=55e7b1ecf9f883aca1c04bd910648430f82c11bba1b1e2db63348f9a623d62d2 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
