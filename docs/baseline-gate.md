# A gate baseline

[The pages](INDEX.md) - [what x3 is](../README.md)

## A gate baseline

**What it catches:** the question *"is this red mine?"*, asked by hand on every
run. Measured in a production repository (2026-09-21): one region verb printed
the same **22 red steps** every time, so the person answering reached for
`git stash` to compare a clean tree — which flips the working tree between two
states and throws the step cache away in both directions. The baseline removes
the reason to touch the tree at all.

The gate reads the same declaration every other subsystem does, so the file
lands beside theirs:

```json
{ "baseline": { "dir": "baselines" } }      # the gate reads baselines/gate.json
```

Declare nothing and **nothing changes**: a gate with no baseline is today's
gate, and every red is red. Adopt one with `x3 gate -update-baseline`, which
writes the red steps standing in the tree right now and never writes growth
afterwards.

A record is keyed by the step's **name**, not by its output. A step's output
carries times, paths and counts, so it differs on every run; a baseline tied to
it would die on the first one. Rename a step and the record dies with it, which
is correct — a step under a new name is not the step whose debt was taken over.

```
$ x3 gate                                   # the frozen red, and one born after it
== the step the baseline froze held (43 ms)
== the step born after the baseline RED (44 ms)
RED: the step born after the baseline
-- baseline: 1 red step(s) carried, 1 born after it, 0 record(s) no longer red   (exit 1)
```

A carried red stays **visible** — it is marked `held`, never hidden. The gate
report is a measurement list, not a findings list: a debt dropped from it could
not be told apart from a step that never ran. When the only red left is carried,
the run exits **0**:

```
$ x3 gate                                   # after the new red is fixed
x3 gate: full band - 2 step(s): 2 ran, 0 skipped, 0 red
-- baseline: 1 red step(s) carried, 0 born after it, 0 record(s) no longer red   (exit 0)
```

A record whose step has turned green is **dead**, and a dead record is red
everywhere in this engine — a stale baseline is how a gate goes blind. It is
printed by name and swept by `-update-baseline`:

```
== the step the baseline froze DEAD (0 ms)
-- baseline: 0 red step(s) carried, 0 born after it, 1 record(s) no longer red   (exit 1)
```

**A narrow run never declares a record dead.** `x3 <region>`, `-only` and any
band below `full` leave most steps unmeasured, and a run that has not measured a
step cannot say its debt is paid; one region verb would otherwise empty the
whole baseline in a single run. The last three counts are the answer to the
question that started this — they stand in the output itself, so nobody has to
reach for the tree to get them.

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
| a finding of a rule the baseline never names | **adopted** — naming it, `ADOPTED <n> of "<rule>"` |
| both at once | the drop lands, the growth stays out, the run still exits `1` |

### A new rule gets the same beginning

The opening balance is **per rule**, for the same reason a missing file is a
beginning: a project that adds a rule to an engine it already uses had no way to
freeze that rule's current violations — they counted as growth, growth is never
written, and the rule was born permanently red. Measured consequence: moving a
check into the engine looked *more expensive* than leaving it in a hand-written
script with its own debt file, which is the opposite of what the baseline is
for.

So a finding whose **rule name has never appeared in the baseline** is adopted
on the next refresh, and said out loud — `ADOPTED 7 finding(s) of "…"`. A rule
the baseline already names cannot grow, exactly as before. Renaming a rule is
not a way around it: the old name's entries go `dead_baseline` red in the same
run.

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

<!-- x3-dist version=v0.276.0 capabilities=b5f0a4a130a3904a44ddab3b5648bead1cce4e08a839e806253889f72d5f3786 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
