# A name that lives outside the list

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -holds`, outside the list


A criterion is not the only thing that calls a test by name. A gate script does
too, and when the test is gone the script quietly runs nothing and still exits
`0`. Those places are **declared**, because no engine can tell which file in a
tree is a gate:

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

**A name the file declares** is reported as `SUSPECT`, verdict `unsure`. A whole
word is a mention, so a short declaration (`lines`) draws lines that only look
like it, and narrowing that would need language knowledge the engine does not
have. The record is kept and printed — never dropped, never called a bond:

```
SUSPECT gates/gate.ps1:2
	name: TestManifestIsWritten
	by: go test ./src/ -run TestManifestIsWritten
	asked: src/manifest_test.go
```

Reading a few extra lines is the price; a wrong "free" is what this refuses to
pay. `places` says how many files were read, and **a declaration that reaches no
file stops the run**: an empty place answers every question with silence, and
that silence cannot be told apart from a declaration that died in a rename.

<!-- x3-dist version=v0.76.0 capabilities=1795187cd09d03d2dc0ea34f2fe0acd59b1733058375cf9b61793cec02fa64ca template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
