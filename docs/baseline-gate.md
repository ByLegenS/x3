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

A record holds one **finding** of a red step, not the step. A baseline that holds
the whole step cannot see a new red born inside it — measured in a production
repository (2026-10-01): the inline-example step was carried red, four new
examples went red inside it, and the region run said `0 born`; the syntax step,
carried with 104 findings, swallowed two new rule violations the same way. Twice
someone read `0 born` as green.

A finding is a line of the red trial's output that opens with a verdict word
(`BLOCK`, `FAIL`, `ERROR`, `panic:`, `--- FAIL`) or carries a file position
(`path.ext:line`), together with the indented lines under it — the rule's reason,
the text it found. `WARN`, `NOTE`, `HELD` and `SELECTED` lines are not findings:
a warning born is not a red. The record is keyed by the step's name, the trial
and that text with **every run of digits folded**, so line numbers, times, counts
and temporary directory names do not move it; two findings that fold to the same
text are told apart by how many times it occurs, so a third one is still born. A
step whose tool prints its findings another way names them itself:

```json
{ "name": "lint", "findings": "^\\s+\\d+:\\d+\\s+error", "trials": [ ... ] }
```

```
$ x3 gate                                   # the held step, with one new finding inside
== the inline examples RED (412 ms)
  1 finding(s) born after the baseline, inside a step it holds:
      BLOCK b.txt:1 (no-shouting): forbidden_pattern
-- baseline: 0 red step(s) carried, 1 born after it (1 of them a held step with a new finding inside), 0 record(s) no longer red   (exit 1)
```

**A baseline written before findings were counted still reads.** Its record holds
the whole step, as it always did, and the run says so instead of keeping quiet:
`-- baseline: N carried step(s) are held whole ...; a finding born inside them
stays unseen until -update-baseline counts them`. The next `-update-baseline`
refines each such record into the findings it was holding (`REFINED`) — the debt
does not grow, it becomes countable. A region verb can do it
(`x3 <region> -update-baseline`): a narrow refresh judges only the steps it
ran and writes every other record back untouched, so one region never empties
another's debt.

**A finding gone from a held step says the baseline can shrink.** A region verb
or `-only` cannot declare a record dead (it did not run every step), but it knows
the steps it measured. A record of one of those steps that this run no longer
finds is named, and the run stays green:

```
$ x3 gate -only "a held step"               # the baseline holds 3 findings, the run finds 2
-- baseline: 1 red step(s) carried, 0 born after it, 0 record(s) no longer red
-- baseline: 1 record(s) are no longer red in step(s) this run measured (a held step); the baseline can shrink: -update-baseline writes it smaller   (exit 0)
$ x3 gate                                   # the unnarrowed run: the same record is dead, red until swept
-- baseline: 1 red step(s) carried, 0 born after it, 1 record(s) no longer red   (exit 1)
```

**An exemption with a reason is not a finding.** A tool that honours
`//x3:allow:<tool>[:<rule>]: <reason>` prints the line it accepted as
`ALLOW <rule> <file>:<line>: <reason>`. That line carries a file position, but the
gate never counts it as a finding: inside a held step it is neither carried nor
born, and the baseline needs no record for it. It does not go quiet either — the
run counts every such line in one summary line:

```
$ x3 gate                                   # a held key, and a second key the source allows
-- baseline: 1 red step(s) carried, 0 born after it, 0 record(s) no longer red
-- 1 allowed: exemption line(s) with a reason, in 1 step(s); not findings, so no baseline holds or counts them   (exit 0)
```

What stays red is what the tool itself reports: remove the exemption line and the
key is born (`BLOCK b.go:4: secret_found`); keep the line after the key is gone and
it is a dead exemption (`BLOCK b.go:5: dead_exemption`); write the line with no
reason and it exempts nothing, so the key is born again. All four arms run in this
engine's own gate (`gate finding baseline control experiment`); the first arm was
red under v0.294.0 (`1 born after it`, exit 1).

**A red step names the findings its printed lines cut off.** The gate prints the
last lines of a red trial; a finding printed higher up (an inline example that
failed, followed by notes and the summary) is named above them:

```
== a held step RED (18 ms)
  the denied word stands in the tree     exit=1 (want 0)
      3 finding(s) above the last lines:
        BLOCK a.txt:1 (no-shouting): forbidden_pattern
        ...
```

Rename a step and its records die with it, which is correct — a step under a new
name is not the step whose debt was taken over.

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
| `-update-baseline` | drop what the run no longer finds; growth is never written. A finding whose identity changed in the same file under the same rule and name is `REKEYED` (paired one to one with the dead record, so the debt per file and rule never grows). A gate red step is never rekeyed: its rule, path and name are all the step's name and its content is already free of numbers, so a new content is a new finding - a held step whose record died and whose run printed a new red gives `GROWTH b215986b7b4b red-step a held step`, the dead record `DROPPED`, no `REKEYED` (through v0.294.0 that dead record handed its identity to any new finding of the step; experiment `baseline refresh only shrinks`, tree `held-rekey`); every other new finding is `GROWTH`, printed by name, and the file is not written for it. A baseline never written and a rule the baseline never named are growth too (until v0.293.0 both were written silently as an "opening balance": `ADOPTED`). Same in `lang`, `arch`, `secrets`, `boxes`, `comments`, `syntax`, `guard`, `placement`, `gate` |
| `-accept-growth` | with `-update-baseline` only: write the new findings as well, each printed as `ACCEPTED <id> <rule> <path> (<name>)`. Example: `x3 syntax -update-baseline` → `GROWTH c9662d70ec83 forbidden_pattern src/new.txt (no-word-beta)` and `the baseline was NOT written`; adding `-accept-growth` → `ACCEPTED c9662d70ec83 ...` (experiment `baseline refresh only shrinks`). Also measured in a pilot: syntax -update-baseline refused 48 visible findings, each by its own `GROWTH` line, exit 1, the baseline file untouched (`git status` clean) - where v0.293.0 had written them silently |

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

<!-- x3-dist version=v0.296.0 capabilities=9df363e2d29d8f8fd6f35424e5d530f5801b7449b9f5f807e7391fd193bde621 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
