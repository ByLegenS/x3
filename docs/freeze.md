# Lists that may only shrink

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 freeze`

**What it catches:** a list that was only ever allowed to get shorter, growing —
the exported surface of a core package, the symbols a binary needs, the debt
somebody promised to pay down.

```
x3 freeze [-config <file>] [-out <file>] [-update] [dir]
```

```json
{ "freeze": { "baselines": [
    { "name": "core-surface",
      "sources": ["internal/core/**/*.go"], "exclude": ["**/*_test.go"],
      "set": { "from": "go", "select": "exported" },
      "file": "baselines/core-surface.json" } ] } }
```

`set` is the same extractor `consistency` uses — `go`, `json` or `regex` — so a
baseline can freeze anything a set can be read from, and it carries the same
`skip`, `comments` and `syntax` fields. `exclude` is read **first** and held to
the same law as everywhere else (`dead_exclusion`, `empty_scope`, `[]` is exit
`2`). Unlike `arch`, a baseline's exclusions are **not inherited**: baselines in
one file rarely read the same tree, and an inherited pattern would be dead — and
therefore red — for the narrow ones.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a value that is not frozen | **red** — `baseline_grew`, named |
| a frozen value that is gone | green, counted as **shrunk** |
| nothing measured at all | **red** — `empty_scope` |

`-update` rewrites the baselines and **refuses to write a set that grew**. That
refusal is the gate: an update that accepted growth would zero the baseline on
every run. Shrink is recorded, because punishing somebody for deleting dead code
teaches people to keep it.

`-update` then **measures again and reports what is left**, so its exit code
means what a plain run's does. Recording a baseline is not the same as passing:
a document above a `cap` is never written anywhere, so an update had nothing to
refuse and used to exit `0` on a tree the plain run called red. An update keeps
its report off stdout; pass `-out` for the post-update JSON.

A missing baseline file measures against an empty set — red until the first
`-update`. A baseline that cannot be measured is `empty_scope`, not a quiet pass.

### The count mode

**What it catches:** a number growing where freezing the *set* of keys would say
only "this document was already on the list", and 1471 lines turning into 1499
would pass in silence.

```json
{ "freeze": { "baselines": [
    { "name": "document-length", "sources": ["docs/**/*.md"],
      "count": { "of": "lines", "min": 1000, "max": 1500 },
      "file": "baselines/document-length.json" } ] } }
```

A baseline writes `set` or `count`, never both.

| `of` | The key | The number |
|---|---|---|
| `lines` | the file | how many lines it has |
| `matches` | the file | how many times `match` occurs, counting **every line** |
| `files` | the directory | how many files it holds |

`min` is where the gate starts looking — without it the baseline would list every
file in the repository. `max` is a **cap that takes no debt**: a key above it is
red whatever the baseline says, and `-update` leaves it out. A cap that could be
frozen would be a request, not a limit. The frozen file carries the numbers, so
a reviewer reads the debt instead of counting it.

| Measured against the baseline | Result |
|---|---|
| a key the baseline does not hold | **red** — `baseline_grew` |
| a number above the frozen one | **red** — `count_grew`, both numbers named |
| a number below the frozen one | green, **shrunk**; `-update` records it |
| a number above `max` | **red** — `above_cap`, never written |
| a key the baseline holds and nothing measures | **red** — `dead_key` |
| nothing measured at all | **red** — `empty_scope` |

The last two are where the modes part: in the set mode a value that is gone *is*
the shrink, but here a key without a number leaves a ceiling standing for a file
that may come back at its old size.

### A floor under a frozen count

**What it catches:** a rule that punishes **smallness**. *A part carrying one
or two files should be rare* — a project splitting a directory into thirty
folders of two files is exactly as unreviewable as the one hundred-and-sixty
file directory it came from. Everything above freezes a ceiling, so that
sentence had no way of being written: the number is allowed to fall to one, and
falling is what the rule forbids.

```json
{ "freeze": { "baselines": [
    { "name": "small-parts-are-rare",
      "sources": ["**/*.go"], "exclude": ["**/*_test.go"],
      "count": { "of": "files", "below": 3 },
      "file": "baselines/small-parts.json" } ] } }
```

`below` is `min` read from the other side: `min` watches the keys **above** a
number, `below` watches the keys **under** one. Writing it turns the frozen
number from a ceiling into a **floor** — and that is the whole declaration.
The direction is not a separate flag, because *"which side am I looking from"*
and *"which way may this number not move"* are one question, and two fields
would let a settings file answer them differently. `min` and `below` together,
or `max` beside `below`, are configuration errors for the same reason.

| Measured against a frozen floor | Result |
|---|---|
| a key under the floor that the baseline does not hold | **red** — `baseline_grew` |
| a number below the frozen one | **red** — `count_fell`, both numbers named |
| a number above the frozen one | green, **shrunk**; `-update` records it |
| a key that rose out of the window | green, **shrunk** — the debt was paid |
| nothing measured at all | **red** — `empty_scope` |

The fourth row is where the two modes genuinely part, and the control
experiment measures it on one tree: a directory whose count went from the
frozen `3` to `4` is `count_grew` under a ceiling and a **debt repaid** under a
floor. Under a ceiling a key that stops being measured is `dead_key`, because a
file may come back at its old size; under a floor, leaving the window is the
only way the debt can be paid, and calling it dead would put a red on the one
outcome the gate exists to produce.

`-update` follows the direction too: it refuses a number that **fell**, and it
drops the keys that left the window. What the run prints follows it as well —
`SHRUNK … above the frozen baseline`, and a refusal that says *"the measurement
is below the frozen one; a frozen floor only rises"*. A sentence that is true in
one mode and false in the other is worse than no sentence: it sends the reader
looking the wrong way.

**What is not here:** a floor with no baseline. `cap` is the ceiling that
freezes nothing; its mirror — *"this directory must always hold at least three
files, and owes nothing"* — is not built.


### The seal

**What it catches:** a line quietly removed from something already published — a
migration that ran in production, a fixture another team pinned to, a contract
file. `set` and `count` cannot say this, and the reason is that both measure
**loosening**: a value that is gone is a shrink, a number that fell is debt
repaid. For a published file there is no such thing as loosening. Adding a line
and removing one break the same promise, and today removing one is green.

```json
{ "freeze": { "baselines": [
    { "name": "published-migrations-are-sealed",
      "sources": ["migrations/*.sql"],
      "seal": { "of": "content" },
      "file": "baselines/migrations.json" } ] } }
```

The baseline holds a fingerprint per file, and the file it lives in is readable:

```json
{ "name": "published-migrations-are-sealed", "count": 2,
  "seals": { "migrations/001_first.sql": "sha256:519a1e8e…" } }
```

| Measured against the seal | Result |
|---|---|
| a sealed file whose content changed | **red** — `seal_broken`, both fingerprints named |
| a sealed file that is gone | **red** — `dead_key` |
| a file the seal does not hold | green — a new migration is not a broken promise |
| the baseline file does not exist | **red** — nothing is sealed yet |
| nothing measured at all | **red** — `empty_scope` |

`-update` **only adds**. It records files the seal does not yet hold, and it
refuses to rewrite a fingerprint that changed or to drop one that vanished — a
seal that rewrites itself seals nothing. Lifting a seal therefore means editing
the baseline file by hand, which is a diff somebody reviews.

The missing-file row is the one that is easy to get wrong. A seal greets an
unknown file with green, so a seal that was never applied would be green
forever; the baseline file's **existence** is the project's statement that it has
sealed something, and its absence is one red until the first `-update`.

**What a seal does not measure:** the bytes on disk. It fingerprints the text the
engine reads, with line endings normalised, because a byte-exact seal would turn
red on a CRLF checkout with the content untouched — and a gate that depends on
how the tree was checked out is switched off the first time it fires. A file
whose *only* change is its line endings keeps its seal.

### A cap with no baseline

**What it catches:** a document that must stay short and owes nothing — a status
page, a handover note. With `count` that sentence cannot be written: putting the
cap above `min` freezes every file between the two at today's size.

```json
{ "freeze": { "baselines": [
    { "name": "status-page", "sources": ["docs/status.md"],
      "cap": { "of": "lines", "max": 300 } } ] } }
```

A baseline writes exactly one of `set`, `count` and `cap`. A key at or below
`max` is green and nothing is recorded; above it is `above_cap` with both
numbers named; nothing measured is `empty_scope`. A cap refuses `file` (nothing
is frozen), `min` (it already reports only what is above `max`) and `policy` (a
limit that can be downgraded to a warning is not a limit). **A cap is always
`block`**, and `-update` cannot reach it — but it must not therefore call the
run green, so an update reports a violated cap like any other run.

<!-- x3-dist version=v0.176.0 capabilities=b557ad04f5f0efc6e52b7370ab28640203fa267a10c6b85b38ee9e370e6da5ff template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
