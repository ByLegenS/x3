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

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
