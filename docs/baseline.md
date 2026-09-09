# Today's findings, frozen

[The pages](INDEX.md) - [what x3 is](../README.md)

## The finding baseline

**What it catches:** the thousand findings a new gate produces on its first run,
which get it switched off by the afternoon. The way out is `freeze`'s, one level
up: write down what the tree owes today, and demand that no new debt appear.

```json
{ "baseline": { "dir": "baselines" } }
```

The configuration declares a **directory**; the file name comes from the
command, so `x3 comments` reads `baselines/comments.json` and `x3 secrets` reads
`baselines/secrets.json`. Declare nothing and there is no baseline: every
finding is red.

| Flag | What it does |
|---|---|
| `-baseline <file>` | read this file instead of the derived one |
| `-update-baseline` | rewrite to this run's findings; growth is never written |

### The identity carries no line number

A baseline keyed by line number moves the day somebody adds an import: the same
finding, the file shifted under it, and a violation nobody introduced. Two of
those and the baseline is refreshed out of irritation. So a finding is
identified by **what it is, where it is, and what it says**:

| Command | A finding is identified by |
|---|---|
| `comments` | the rule and the file |
| `secrets` | the rule, the file, the pattern and the **masked** sample |
| `arch` | the code, the file, and the rule, subject and object |
| `lang` | the code, the file, and the token with the place it sits in |
| `syntax` | the code, the file and the name of the check |

```json
{ "version": 1, "command": "comments", "count": 2,
  "findings": [ { "id": "4ace75caf575", "rule": "block_too_long", "path": "a.go" } ] }
```

The **digest**, not the text, is what the run compares — a baseline storing the
matched text would put the very value `secrets` masks into a file the repository
keeps.

### The direction is the whole point

| Measured against the baseline | Result |
|---|---|
| a finding the baseline holds | green, counted in `baselined` |
| a finding the baseline does not hold | **red** — this is the gate |
| a baseline entry the run no longer produces | **red** — `dead_baseline` |

`-update-baseline` **refuses to write a set that grew**, naming every finding
that blocked it; without that refusal the flag would turn any red run green. A
missing baseline file measures against an empty set; a file that exists and
holds nothing is a project declaring it owes nothing, and can only shrink. **An
empty file is a statement, a missing file is a beginning.**

### What can never enter a baseline

- **Warnings** — an observation is not a debt, and freezing one turns into a
  `dead_baseline` red the moment somebody fixes it.
- **Scope-integrity findings** — `empty_scope` says the rule measured nothing;
  freezing it paints a gate that checks nothing green.
- **Dead markers** — `dead_exemption`, `dead_exclusion`, an uninstalled parser.
  They belong to the gate's own health, not to the source.

<!-- x3-dist version=v0.65.0 capabilities=30f2211593ea62df95d9a529b650866118e447096978014873bc8ee488525447 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
