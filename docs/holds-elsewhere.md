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

Every line of those files is read as text, and a line containing an asked name is
reported as a `USED`, counted with the holds:

```
USED  gates/gate.ps1:2
	name: TestManifestIsWritten
	by: go test ./src/ -run TestManifestIsWritten
	asked: src/manifest_test.go
```

`places` in the summary says how many such files were read. **A declaration that
reaches no file stops the run**: an empty place answers every question with
silence, and that silence is indistinguishable from the declaration having died
when a directory was renamed.

The place is read as plain text and a whole word is a use, so a short declaration
(`lines`) draws lines that only look like it. Narrowing that would mean knowing
which of a file's declarations a script can even call — language knowledge the
engine does not have, and guessing it would make the mode quietly miss the
languages it does not recognise. Reading a few extra lines is the price; a wrong
"free" is the thing this refuses to pay.

<!-- x3-dist version=v0.75.0 capabilities=24f285e04e908d4e7cfcd4ff0c98505f172d6cc2944b582a432f8bd6ac27f89d template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
