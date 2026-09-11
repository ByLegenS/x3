# The names on a gate's line

[The pages](INDEX.md) - [what x3 is](../README.md)

## The names on a gate's line

A gate line carries names as well as paths, and they are not all worth the same.
Two classes are separated here, and only one of them is a bond.

**A name the file declares**, standing on the line as a word, is reported as
`SUSPECT`, verdict `unsure` — but only a name an outsider could write. A
declared place sits outside the package, and Go itself says what can be named
from there: an **exported, package-level** declaration. An unexported name
cannot be written from outside at all, and a method name alone points at no
declaration — in both classes the match is the word, not the symbol. `bindable`
counts the names that survived the question. There is nothing to weigh a bare
word against, so it stays `unsure`.

**The selector the line hands its runner** is the other class, and it is a bond.
A gate step usually names neither the file nor the whole test — it hands the
runner a *fragment*: `-run Manifest`. No word search finds
`TestManifestIsWritten` on that line, so the file looks free; delete it and the
runner matches nothing, prints `no tests to run` and still exits `0`. The engine
cannot know which argument is the selector — a runner's flag name in the engine
would mean one runner recognised and the next one not — so the **shape is
declared**, one capturing group, and the capture is compiled as a pattern and
matched against the names the file declares:

```json
{ "boxes": { "holds": {
  "sources": ["check.ps1", "gates/**/*.ps1"],
  "selects": ["-run\\s+([^\\s'\"]+)"]
} } }
```

```
USED    gates/gate.ps1:3
	name: TestManifestIsWritten
	by: go test ./service/ -run Manifest
	asked: service/manifest_test.go
```

A pattern knows no place, so **the line's own path tokens narrow it**: the same
`Manifest` selector matches a test of that name anywhere in the tree, while the
line runs one package. A line that names no place runs from the root and reaches
everything — the same reading a command criterion gets
([Which hold is really a hold](holds-weight.md#which-hold-is-really-a-hold)). `selectors` counts
what was read, a reading with anything but one capturing group is a
configuration error, and **a reading that reaches no line stops the run**: a
pattern that finds nothing answers every question with silence, exactly as an
empty `sources` does. A captured fragment that is not a valid pattern is skipped
— the line is the project's prose, not its configuration — and the word match
still applies to it.

⚠️ **A place that names the whole tree is not evidence about one file.** A gate
that runs every test in the repository runs *this* file too, and saying so about
every file distinguishes none of them; deleting a test there leaves the step
green and measuring less, which is a different question from the silent zero
this command exists to catch. Only a selector aimed at the name, or the file
written out as a path, is read as a bond.

<!-- x3-dist version=v0.97.0 capabilities=e600efd1d92bff25f5816e9f3d989f80c770f098167ab364e947e723d56fa25d template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
