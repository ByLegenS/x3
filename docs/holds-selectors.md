# What a gate's line holds

[The pages](INDEX.md) - [what x3 is](../README.md)

## What a gate's line holds

A gate line carries names and paths, and they are not all worth the same. Three
classes are separated here, and two of them are bonds.

**A name the file declares**, standing on the line as a word, is reported as
`SUSPECT`, verdict `unsure` — but only a name an outsider could write. A
declared place sits outside the package, and Go itself says what can be named
from there: an **exported, package-level** declaration. An unexported name
cannot be written from outside at all, and a method name alone points at no
declaration — in both classes the match is the word, not the symbol. `bindable`
counts the names that survived the question. There is nothing to weigh a bare
word against, so it stays `unsure`.

**The selector the line hands its runner** is a bond. A gate step usually names
neither the file nor the whole test — it hands the runner a *fragment*:
`-run Manifest`. No word search finds `TestManifestIsWritten` on that line, so
the file looks free; delete it and the runner matches nothing, prints `no tests
to run` and still exits `0`.

**The package the step runs** is the third class, and the widest. `go test
./service/` measures every test file in that package and writes down none of
them: no path token points at `service/manifest_test.go`, no selector is handed
over, and the file reads free — delete it and the step still exits `0`,
measuring one thing less.

Neither shape is guessed. A runner's flag name inside the engine would mean one
runner recognised and the next one not, so both are **declared** — `selects`
with exactly **one capturing group**, compiled as a pattern and matched against
the names the file declares, and `runs` with **none at all**, because it only
marks the line and the places come from the line's own path tokens:

```json
{ "boxes": { "holds": {
  "sources": ["check.ps1", "gates/**/*.py"],
  "selects": ["-run\\s+([^\\s'\"]+)"],
  "runs": ["\\{ go test ", "\"go\", \"test\","]
} } }
```

```
USED    gates/gate.ps1:3
	name: TestManifestIsWritten
	by: go test ./service/ -run Manifest
	asked: service/manifest_test.go
USED    gates/gate.ps1:7
	name: service/manifest_test.go
	by: Step "smoke" { go test ./service/ -count=1 }
	runs: service
```

A selector knows no place, so **the line's own path tokens narrow it**: the same
`Manifest` matches a test of that name anywhere, while the line runs one package
— the reading a command criterion already gets
([Which hold is really a hold](holds-weight.md#which-hold-is-really-a-hold)). `selectors`,
`runs` and `scopes` count what was read, and **a reading that reaches no line
stops the run**, exactly as an empty `sources` does. A captured fragment that is
not a valid pattern is skipped — the line is the project's prose, not its
configuration — and the word match still applies to it.

⚠️ **A step that names the whole tree holds nothing, and the run says so.** A
gate running every test in the repository runs *this* file too, and saying that
about every file distinguishes none of them; deleting a test there leaves the
step green and measuring less, which is a different question from the silent
zero this command exists to catch. Measured on a real production Go application:
of the 213 files no criterion held, **213** sat under some line naming a place,
so a reading that took every place would have emptied the pool. The root is
dropped from a running step, and a declaration whose marked lines name nothing
narrower stops the run instead of answering every question alike. With the steps
that do name a package: **213 free → 187**.

<!-- x3-dist version=v0.140.0 capabilities=b8445c1227f160e93100c6d30aa776467b63ccee66c2b1d4455c262b73b1ed73 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
