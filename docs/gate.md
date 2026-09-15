# The gate itself, running in parallel

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 gate`

```
x3 gate [-config <file>] [-band fast|commit|full] [-only <names>] [-workers <n>] [-tune] [-out <file>]
        [-changed] [-red-exit <n>]
```

**Catches:** a gate that lives in a shell script — which every project then writes
again, and which runs its steps one after another because a script has no other
way to run them.

A gate is a list of things to run and the exit each one should give. That is a
**list**, not a program, and once it is written as a list two things follow: the
engine can run it, and it can run the steps **at once**. Most steps touch nothing
the others touch — each gets its own working directory — so the order a script
imposes is the script's limit, not the work's.

```json
"gate": {
  "binary": "_build/tool.exe",
  "build": "./cmd/tool",
  "stamp": "main.version",
  "steps": [
    { "name": "gofmt", "band": "fast",
      "trials": [ { "say": "every file formatted", "run": "gofmt -l .", "want": 0, "not": ".go" } ] },
    { "name": "language gate", "band": "fast",
      "trials": [
        { "say": "this repository", "run": "{bin} lang .", "want": 0 },
        { "say": "a planted sample", "run": "{bin} lang testdata/red", "want": 1 }
      ] }
  ]
}
```

### The steps run at once — and so does everything they call

**Measured in a production Go application on 32 logical cores:** the gate ran 32
steps at once, and each step's own tools spread across all 32 cores as well —
32 × 32 = 1 024 threads on 32 cores. The processor sat at **99.6% busy** and the
work still finished late: 44 compiler processes taking turns on 32 cores, each
one hauling its own data back into the cache on every turn. Two layers of
parallelism, neither aware of the other.

`gate.env` declares what every step's environment carries, and `{cores}` opens to
one step's share of the machine: processors ÷ workers, never below one.

```json
"gate": { "workers": 24, "env": { "GOMAXPROCS": "{cores}" } }
```

| The same tree, the same steps, nothing removed | Wall | Total work |
|---|---:|---:|
| 32 workers, no budget | 1:58 | 2 012 s |
| 24 workers, `{cores}` | **1:49** | **1 558 s** |

⛔ **The engine names no tool.** It hands out the share as `{cores}`; which
variable carries it is the project's word (`GOMAXPROCS`, `MAKEFLAGS`, …), and a
project that wants a flat number simply writes one. A step's own `env` and a
trial's `env` write over this one, in that order.

**Busy is not useful.** The measured difference between a share of 1 and a share
of 2 was noise (1:49 ↔ 1:47), so the division is enough; a separate ratio field
would only be one more number to get wrong.

### The worker count is the machine's, not the project's

The peak sits in a different place on every machine: processors, disk, database
and the mix of steps all move it. A number found by hand once becomes a number
nobody dares touch, and the day the machine changes it is quietly wrong.

`-tune` runs the gate at four worker counts — the processor count and its three
quarters, half and quarter — and reports the peak:

```
x3 gate -tune
x3 gate: measuring this machine's worker count - 4 full run(s)
x3 gate: 24 worker(s) on 32 processor(s) - 32:118204ms 24:108094ms* 16:121530ms 8:186402ms
```

The result is written **beside the cache, not into the settings**: a worker count
is a property of the machine, and a machine's number pushed into a shared file is
wrong for everyone else who reads it. The record carries the processor count that
produced it, so a binary copied to another machine — or a virtual machine that
grew — measures again instead of trusting a number that no longer describes
anything.

**A gate with no declared `workers` tunes itself.** There is no command to
remember: an absent setting is what triggers the measurement, a stored one is
read, and `-tune` forces a fresh measurement over both.

⛔ **The measurement checks itself.** If one point leaves a different number of
steps red than the others, the points did not do the same work and their times
cannot be compared — a red step exits without finishing, which shortens the wall
clock and flatters the point that broke. Measured in a production application: the
same tree run without its database declared reported 18 red in 71 s, and with it
11 red in 111 s. A tuning run like that writes nothing and **names the steps that
moved** — a count alone sends the reader back to run the gate by hand twice to
find out which one is unsteady, and that name was already in the report. Two
points that leave the same number red but not the same steps are just as
incomparable, and are reported the same way. A tuning run like that says why — and then
**the gate runs anyway**, one worker per processor. A measurement is a
convenience, never a precondition: the opposite would leave a project whose gate
is red unable to run the gate that shows it. `-tune`, asked for on purpose, does
report the failure and exits 2.

### A step that cannot have changed is not run again

```
x3 gate [-cache <file>] [-no-cache]
```

Most of a mature gate measures **the gate itself**: steps that build a made-up
tree, run the engine against it and check what it says. Measured in a production
application: of 254 trials, **169 ran in a made-up tree** and cost 516 seconds of
every run — 55% of the total. Those trials cannot see the repository's own code,
so a commit to it cannot change their answer. They depend on the engine's version
and on the settings, and both are already in the cache's salt.

A step is remembered only when **every** trial in it carries a `tree`. One trial
reading the real repository puts the whole step outside the cache: a step is one
thing, and half of it remembered would be a gate reporting on work it did not do.

⛔ **Only green is stored** — the same law the rest of the engine follows. A red
step enters no cache, so a fix is always measured, and a fault that is still there
is never hidden by a memory of the day it passed.

### A step whose files did not change

```json
{ "name": "whatsapp worker", "touches": ["apps/whatsapp/**", "core/**"], "trials": [...] }
```

A step that declares what it reads is skipped when none of it changed in this
run. This is the same law as the step cache, applied to the steps that do look at
the repository: one reads a made-up tree and is skipped when the **settings** are
unchanged, the other reads the real one and is skipped when **its own files** are.

⛔ **A step that declares nothing runs every time.** The engine does not guess
what a step reads: a wrong guess shows something green that was never measured,
and it does so in silence. Writing `touches` is the project's decision, and it
documents the step's scope in the same line.

⛔ **The change set is read once**, not per step — `git status` per step is one
process per step. If git cannot be read the list stays empty and no step is
skipped: a gate that cannot see its own input does not fall silent, it runs.

### A hook can run the gate at the end of every turn

`x3 gate -band fast -changed -red-exit 2` — two flags for callers that are not a
person reading the output. `-changed` measures nothing when the working tree is
clean, so a hook that fires every turn costs nothing on the turns that wrote no
files; when git cannot be read the gate **runs anyway**, because a gate that
cannot read its own input must not go quiet. `-red-exit <n>` picks the code a red
step returns (1 by default): hook protocols differ, and without the flag a
project had to wrap the gate in a script that only translated one number into
another.

### Overlays can live in one ledger

`-with <file>` takes an overlay. `-with <file>:<name>` takes one **out of** a
ledger — a file whose top-level keys are overlays. One file per overlay measured
32 files in a production repository, averaging fifteen lines each; thirty-two
files in a directory are not a list, they are a pile, and comparing two of them
meant opening two files. A name the ledger does not carry stops the run.

### A trial can run on a planted tree

A control experiment needs a small wrong tree to measure against. Keeping those
trees on disk looks harmless and is not: measured in a production repository,
nineteen experiments held **420 files of which only 140 were unique** — 67% was
one limb copying another limb's `go.mod`. A copy goes stale the day the base
changes, and four hundred files are not read, they are scanned.

`gate.trees` declares them instead. A tree has a **base** and **limbs**; a limb
writes only what differs, and `null` removes a file the base laid. The trial
names `"<tree>:<limb>"`, the engine plants it before the call and removes it
after, and `{tree}` is where it stands.

```json
"trees": {
  "sample": {
    "base":  { "x3.yaml": "{ … }", "data.json": "{ \"ok\": true }" },
    "limbs": { "green": {}, "broken": { "data.json": "{ \"ok\": tru" },
               "pruned": { "data.json": null } } } },
"steps": [
  { "name": "planted tree", "band": "fast", "trials": [
    { "say": "the base alone parses", "tree": "sample:green",
      "run": "{bin} syntax -config {tree}/x3.yaml {tree}", "want": 0 },
    { "say": "a limb that breaks a file", "tree": "sample:broken",
      "run": "{bin} syntax -config {tree}/x3.yaml {tree}", "want": 1 } ] } ]
```

A limb nobody declared, and a `null` over a file the base never laid, are both
settings errors — a limb quietly running on an empty directory measures nothing.

**`run` takes a line or a list.** A line is split the way a shell splits one:
spaces separate, and a quoted span (`"` or `'`) stays one argument. A list is
taken as written. Measured in a production repository before this was allowed:
of the gate's 3 264 lines, **1 929 were the `run` array alone** — 210 calls, each
spread over nine lines. A gate list is read; a call spread over nine lines is
scanned, not read. The list form stays, because it is how an argument carrying a
space is written without quoting, and how generated settings write it.

`{bin}` is the binary the gate builds before any step runs — built first on
purpose, because steps run in parallel and one of them producing the file the
others call would be a race. `{tmp}` is the step's **own** working directory, so
two steps writing "the report" never write the same path.

### Every trial says what it expects

`want` is not optional. *"It ran and did not explode"* is not a measurement, and a
green step means something only when the red beside it is really red — which is
why a step carries several trials: the clean tree exits `0`, the planted one exits
`1`, and both are printed. `says` and `not` measure the output the same way for a
command whose exit code alone cannot tell the two apart.

### The ladder, and what is skipped

`band` puts a step on the ladder `fast < commit < full`; asking for `commit` runs
`fast` too, because a commit run that skipped the quick steps would measure not
less but **wrong**. `needs` names an environment variable a step cannot run
without. A skipped step is always **printed with its reason**: a step that goes
quiet is a hole in a green run that nobody can see.


### A red the step can live with

Some reds are not the tree's fault: the tool is not installed on this machine,
the vulnerability database could not be reached. A gate written as a script
handles those with a branch that reads the output and downgrades the colour —
and every project writes that branch again, differently.

```json
{ "run": ["govulncheck", "./..."], "want": 0,
  "tolerate": ["loading vulnerability database", "executable file not found"],
  "because": "the network's failure is not this repository's red" }
```

`because` is required. A tolerated red nobody explained is never questioned
again, and the difference between "the database was unreachable" and "the
check has been off for a month" is exactly that sentence.

### The variables a step declares

`needs` says a step cannot run **without** a variable. `env` says what the step's
own environment **is**:

```json
{ "name": "schema drift", "band": "full",
  "env": { "APP_DEV_DSN": "${APP_DSN}" },
  "trials": [ { "run": ["{bin}", "guard", "-only", "schemadrift"], "want": 0 } ] }
```

A trial can carry `env` of its own, and it wins over the step's: one command,
two arms, two environments. `${NAME}` also opens **inside** a value
(`"sandbox;${PATH}"`), and `${NAME:-fallback}` gives an unwritten variable a
default that lives in the settings rather than in a script. The same opening
happens in a command's **arguments**: a step whose target changes from machine
to machine carries its default where the step is read.

One address, two names, and both of them pointing at the same place in the same
run — otherwise the same tree gives two different numbers in two gates. A shell
script does this with a wrapper function around every step that shares a database;
a list does it in the step itself, where it is read next to the step it belongs to.

A declaration whose source is **empty** is red and the command does not run at
all: a variable that quietly arrives blank is a step measuring something other
than what it says. A value that is not `${...}` is used as itself.

### What parallel buys, measured

The report prints the wall clock next to the work done — `1.5x faster than one
after another (20476 ms of work)` — and the five slowest steps with their share.
Both numbers are there for the same reason: a gate nobody can see inside of is a
gate nobody makes faster, and a single total hides the one step eating the run.

<!-- x3-dist version=v0.239.0 capabilities=10d9c3c1d915b27d22dbf343c4ebaa44b17ab60bb6073a976b4b7c42d799f084 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
