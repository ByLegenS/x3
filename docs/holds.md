# Before a name is removed

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 boxes -holds`

**What it catches:** a criterion that keeps passing after the thing it measured
was deleted.

Criteria lean on names in the tree — a declaration, a path, the selector handed
to a runner. Remove one and a criterion can stop measuring without turning red:
a selector matching nothing makes most runners exit `0`, and `0` reads as
"done". A silent green is worse than a red one, because nobody looks at green.

```
x3 boxes -holds internal/parse/line_test.go
x3 boxes -holds-from removing.txt
```

Each entry is a **name or a path**, `-holds` comma separated and `-holds-from`
one per line (`#` opens a comment). A path that exists on disk is also asked as
every top-level name that file **declares**: what gets removed is usually a
file, while the name a criterion holds is written inside it, and leaving that
step to a script outside the engine is what this mode exists to end.

**Nothing is run.** The question is not "does the criterion hold" but "does the
criterion lean on this name", and it has to be answerable **before** the
removal, not after it.

| `how` | The criterion | A search would |
|---|---|---|
| `text` | writes the name in its own words — `match`, `path`, `sources`, `query`, an argument | find the line, but not which box it belongs to, nor whether that box is open |
| `pattern` | hands a **selector** to a runner, and the selector read as a regular expression matches the name | **not find it at all**: a selector may be a fragment of the name |

The second row is why the question is asked backwards, pattern against name
rather than name against text. A criterion running `-run ParseLine` proves
`TestParseLineKeepsBlanks`; searching the documents for that test's name returns
nothing. Which argument is the selector is **not guessed** — the kind declares
it (`markdown.criterion.kinds[].batch.select`), and no runner's flag name ever
enters the engine. Without that declaration the `pattern` row is silent, which
is the honest answer rather than a convenient one.

```
HOLD  WORK.md:197005faf068: pattern
	name: TestTheGatedWorkIsProven
	box: the gated work a running test proves
	by: command go test -v ./... -run TheGatedWork
	at: WORK.md:3
	asked: src/gated_test.go
x3 boxes: 1 asked, 2 name(s) - 1 held, 0 free - 3 criterion(s) in *.md
```

Red when a name is held, green when none is. The summary counts the criteria it
read, so a green answer from a list carrying **no** criteria can be told apart
from a green that measured something.

<!-- x3-dist version=v0.70.0 capabilities=d6bca49b1ee20fb16cf56855193fb72748bc6213792c4f4e81682cf9ef31d4b0 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
