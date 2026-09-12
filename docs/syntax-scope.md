# What a check does not read

[The pages](INDEX.md) - [what x3 is](../README.md)

## What a `syntax` check does not read

A source glob describes a **tree**, and a check's real subject is often a tree
*minus* a handful of paths: the four screens behind an administrator's door,
the ledger's own type, the file written to disk and never sent anywhere. Stars
cannot write a subtraction, so the scope had to be widened until it was wrong
and the gate turned noisy — and a noisy gate is a closed gate.

```json
{ "syntax": { "checks": [
    { "name": "no-margin-word-reaches-the-browser",
      "sources": ["app/**/*.go"],
      "exclude": ["**/*_test.go", "app/ledger/ledger.go"],
      "deny": "cost_micros", "reason": "the margin is ours, not the tenant's",
      "comments": "exempt" } ] } }
```

`exclude` removes paths the `sources` glob took, and its name and reading are
the ones an `arch` rule already carries. It is not the third escape hatch of
this check but the first of three, each on its own axis: **`exclude` drops a
file**, `ignore` drops a **line** the pattern caught, and `holds`/`lacks` drop
a file by what it **contains**.

### Prose is not a violation

`comments: "exempt"` stops the denied pattern from reading comment text. The
sentence that explains *why* a word is forbidden has to write that word:

```go
// cost_micros is ours; a tenant-facing endpoint never returns it.
```

Counting that as a violation rewards deleting a correct explanation, and it
does worse than that — it produces confident wrong diagnoses. Measured on a
real production Go application: of the findings a first port of one gate
raised, the ones left after the paths were excluded were **entirely** comments
explaining the rule, and reading them as code had already turned into *"we
found six violations"* when there were none.

Comments are **blanked, not deleted**, so line numbers stay where they are and
the finding names the real line. The default is `checked` — the reading every
configuration so far was measured under, and a default that changes quietly is
a rule nobody declared.

### The laws

- Both belong to the check they are written on, not to the section.
- `comments` needs `deny`. A parser reads the file **from disk**; a blanked
  copy never reaches it, so the declaration would measure nothing. The same is
  already true of `ignore` and of `directives`.
- `syntax` replaces the comment markers for an extension, and it needs
  `comments: "exempt"` beside it, or it changes nothing.
- An **unknown extension is read as all code**. Not knowing costs in one
  direction or the other, and this one is chosen: a check that reads too much
  shouts, a check that reads too little agrees with everything.
- An `exclude` that empties the whole set is `empty_scope` and red, the same
  as a `sources` glob that matches nothing.

<!-- x3-dist version=v0.149.0 capabilities=aa2b3d5359a52c0465529a4d78500da0ece5c1d342d9261b163f39c08cf09ce1 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
