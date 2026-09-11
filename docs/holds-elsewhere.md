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

`places` says how many files were read, and **a declaration that reaches no file
stops the run**: an empty place answers every question with silence, and that
silence cannot be told from a declaration that died in a rename.

What the engine does with the **names** on those lines is a second question —
[The names on a gate's line](holds-selectors.md#the-names-on-a-gates-line).

<!-- x3-dist version=v0.93.0 capabilities=8e6ea0dead868abe1ef78d99ea4880c63abef385ce80ffc483b9f1a56eb54532 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
