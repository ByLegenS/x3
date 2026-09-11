# A name that lives outside the list

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -holds`, outside the list


A criterion is not the only thing that calls a test by name. A gate script does
too, and when the test is gone the script quietly runs nothing and still exits
`0`. Those places are **declared** — no engine can guess which file is a gate:

```json
{ "boxes": { "holds": { "sources": ["check.ps1", "gates/**/*.ps1", ".github/workflows/*.yml"] } } }
```

Every line is read as text, and two different things are found there. **The file
itself, written as a path**, is a bond — a person put it there — `USED`, `held`:

```
USED    gates/gate.ps1:4
	name: src/golden.txt
	by: Get-Content src/golden.txt
```

**A name the file declares** is reported as `SUSPECT`, verdict `unsure` — but
only a name an outsider could write. A declared place sits outside the package,
and Go itself says what can be named from there: an **exported, package-level**
declaration. An unexported name cannot be written from outside at all, and a
method name alone points at no declaration — in both classes the match is the
word, not the symbol. `bindable` counts the names that survived the question:

```
SUSPECT gates/gate.ps1:2
	name: TestManifestIsWritten
	by: go test ./src/ -run TestManifestIsWritten
	asked: src/manifest_test.go
```

`places` says how many files were read, and **a declaration that reaches no file
stops the run**: an empty place answers every question with silence, and that
silence cannot be told from a declaration that died in a rename.

<!-- x3-dist version=v0.90.0 capabilities=ddd6422e823fdef53eb890d125f81c56bb91f8d7d1d213804289b7a90ff40f1d template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
