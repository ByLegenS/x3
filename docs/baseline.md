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

Seven commands carry one: `arch`, `comments`, `secrets`, `lang`, `syntax`,
`boxes` and `guard`. The first six measure the **state of the tree**, which is
what a debt is. `guard` measures a running system instead, and a red there is a
debt of the same shape: *"this check does not hold today, and no **new** one may
appear"*. `docs` and `scope` read a diff, and a finding there is not debt but
the change in front of you; freezing it would silence the wrong thing.

| Flag | What it does |
|---|---|
| `-baseline <file>` | read this file instead of the derived one |
| `-update-baseline` | drop what the run no longer finds; growth is never written |

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
| `guard` | the guard's name, the kind of red, and the **expectation** |

```json
{ "version": 1, "command": "comments", "count": 2,
  "findings": [ { "id": "4ace75caf575", "rule": "block_too_long", "path": "a.go" } ] }
```

The **digest**, not the text, is what the run compares — a baseline storing the
matched text would put the very value `secrets` masks into a file the repository
keeps.

`guard` has no file to key on, and what it observed is the one thing that cannot
be part of the identity: a row count, a status code, a version string - it moves
between runs, and a debt keyed on it would die on its first flutter, shouting a
new finding and a `dead_baseline` in the same breath. The expectation is the
opposite case: it **is** in the identity, so quietly rewriting one does not
inherit the frozen record - the old debt dies loudly and somebody reads it
again.

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

### The two directions are refreshed separately

Growth and shrink are not two halves of one decision, and joining them punishes
the wrong person. A tree usually carries both at once: a file that owed
something was deleted, and somewhere else a new finding appeared.

| In one run | What `-update-baseline` does |
|---|---|
| a finding the baseline does not hold | never written — naming it, `GROWTH <id>` |
| an entry the run no longer produces | **always dropped** — naming it, `DROPPED <id>` |
| both at once | the drop lands, the growth stays out, the run still exits `1` |

Dropping an entry can only make the gate **stricter** — the debt it excused is
gone, so nothing can hide behind it — which is why it needs no permission from
the other direction. Joined, the opposite happened: one unrelated finding froze
the whole file, every deleted file left a permanent `dead_baseline` red, and the
only way out was editing the JSON by hand.

`count` is **derived** and a file disagreeing with its own list is refused with
exit `2`; see [A baseline two branches write](baseline-parallel.md#a-baseline-two-branches-write).

### What can never enter a baseline

- **Warnings** — an observation is not a debt, and freezing one turns into a
  `dead_baseline` red the moment somebody fixes it.
- **Scope-integrity findings** — `empty_scope` says the rule measured nothing;
  freezing it paints a gate that checks nothing green.
- **Dead markers** — `dead_exemption`, `dead_exclusion`, an uninstalled parser.
  They belong to the gate's own health, not to the source.

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
