# What this project asks of the engine

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 config`

```
x3 config [-config <file>] [-out <file>] [-check]
```

**Catches:** a project whose own tooling nobody can see. A mature repository
declares a dozen capabilities across a dozen settings files, and the answer to
"what do we actually run here, and what does it cover?" lives in nobody's head.

`x3 config` writes that answer as settings, not prose: the engine's version, the
shortcuts `x3 do` offers with their flags, the capabilities this project declares
and the ones it leaves idle, and every gate step with its band, what it reads,
and whether it can be remembered between runs.

⛔ **Three lists, not two.** A command with no settings section of its own is
neither declared nor idle — `fmt` runs every turn and asks for nothing. Written
as "not used" the inventory reports a gap that is not there, and the reader goes
off to close it.

### A long line nobody types twice

```json
{ "alias": { "full": "gate", "quick": "gate -band fast", "suite": "test -fresh-db" } }
```

`x3 quick` runs `x3 gate -band fast`. The caller's own arguments are appended, so
`x3 quick -out report.yaml` works and adds to the alias rather than replacing it.

A command line that is not remembered is either typed wrong or not typed at all.
Measured in a production application on the day this was written: a gate run
without its database variable declared reported **18 red and a false 71 seconds**
— the line itself was right; what was missing was writing all of it every time.

⛔ **An alias cannot shadow a real command.** The table is consulted only for a
name the engine does not know, so `x3 gate` is always the engine's own gate even
if a project writes an alias by that name. Letting settings rename a tool's own
commands means the same command doing two different things in two repositories.

⛔ **It is binding, not decorative.** `-check` writes nothing and exits 1 when the
file on disk is not what this run would write. As a gate step it means a change
to the settings cannot land while the inventory still describes yesterday's
project. An inventory nobody verifies is worse than none: it is read with trust
and it is wrong.

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

A step is remembered when the engine can name everything it reads: either **every**
trial carries a `tree` (nothing in the repository reaches it), or the step declares
its files in `touches` and their contents go into the key. One trial reading a
repository the step never declared puts the whole step outside the cache — a step
is one thing, and half of it remembered would be a gate reporting on work it did
not do.

⛔ **The key carries the binary the gate measures with.** A gate that builds its
own tool from the repository (`gate.build`) would otherwise remember a step across
an edit to that tool: the version stamp is a git tag and does not move when a
source file does.

⛔ **Red is stored too, and comes back red.** An earlier version kept only green,
reasoning that a stored red repeats a sentence this run never measured. The same
is true of green — so that rule was defending the *key*, not the colour. Once the
key carries the content of everything the step reads, both colours are equally
sound: the inputs did not change, so neither did the result. Measured in a
production application: two full runs back to back on an untouched tree ran the
same 57 steps twice (59 578 ms, then 57 200 ms, 9 red both times). Every one of
those seconds bought an answer that was already known.

⛔ **A remembered red is counted red.** The summary looks at the colour of every
step, not only of the ones that ran — otherwise a fault would go quiet on the
second run, which is the one failure this whole feature could cause.

### One step written once, run per application

```json
{ "name": "vet · {each}", "each": { "dirs": "apps/*" },
  "touches": ["{each}/**", "core/**"],
  "trials": [{ "run": "go vet ./{each}/...", "want": 0 }] }
```

A repository with three applications should not carry three copies of the same
step, and one with a thousand should not carry a thousand. `each.dirs` matches
directories on disk and the step is produced once per match, with `{each}` opened
to that directory in the name, the commands, the environment and the scope.

**The list lives on disk, not in the settings.** A new application is measured the
day it is created; nobody has to remember to copy a step, and no copy goes stale
on its own.

⛔ **Each copy carries its own scope.** Written with `{each}` in `touches`, a copy
reads only its own tree — so a change to one application does not run the other's
steps. That is where the saving is; multiplying alone would only shorten the
settings file.

⛔ **A pattern that matches nothing is an error**, not a silent gap: a dead
pattern removes a step from the gate without anyone noticing, and a tree nobody
measures is exactly what a gate exists to prevent.

### A step whose files did not change

```json
{ "name": "messaging worker", "touches": ["apps/messaging/**", "core/**"], "trials": [...] }
```

`touches` is the step's cache key: the content of every file it names is hashed
into the key, and the step runs only when one of them changed since it last ran.
This is the same law as the step cache, applied to the steps that do look at the
repository: one reads a made-up tree and is remembered while the **settings** hold
still, the other reads the real one and is remembered while **its own files** do.

⛔ **The question is "since this step last ran", not "since the last commit".** An
earlier version asked git what had changed and skipped the steps that did not match
— which answers a different question. Switching branches, committing, or doing
nothing at all moves git's answer without moving a step's input. Worse, a skip like
that had to *assume* a colour, and it assumed green: a step that was red, whose
files were not in the last commit, came back green without being measured. Content
in the key removes the guess — the step returns the result it actually produced.

⛔ **A step that declares nothing runs every time.** The engine does not guess
what a step reads: a wrong guess shows something green that was never measured,
and it does so in silence. Writing `touches` is the project's decision, and it
documents the step's scope in the same line.

### One scope for the whole gate, and the steps that cannot have one

```json
{ "gate": { "reads": ["apps/**", "core/**", "settings.json"],
            "steps": [ { "name": "ledger ↔ production", "volatile": true, "trials": [...] } ] } }
```

`gate.reads` is the scope every step inherits when it declares no `touches` of its
own; a step's own declaration replaces it. Measured in a production application:
of 73 steps only **5** declared a scope, because declaring one meant repeating the
same twelve roots 68 times — so 68 steps re-ran on every gate, and two consecutive
runs on an untouched tree measured 57 steps twice. The cost of repeating yourself
was being paid in seconds.

⛔ **A scope has to be honest, not narrow.** A list naming every measured root of
the repository is already enough to answer "nothing changed at all", which is the
case that costs the most. Narrowing it — this step reads only this application —
is a second, separate gain, made one step at a time and measured.

### A step whose input is a database

```json
{ "name": "ledger reconciliation",
  "state": "x3 testdb state -env APP_DSN",
  "trials": [...] }
```

`state` names a command that **prints** the step's non-file input; its output goes
into the key. The step is remembered while that line holds still and runs again
the moment it moves. The engine does not know what is being asked — the project
writes the command, so a project on another kind of store writes another command.
`x3 testdb state` is the one the engine ships: one line summarising the database,
measured at **1.4 s** against a remote server, in place of steps that cost 76 and
78 seconds every run.

⛔ **A state command that fails leaves the step unremembered.** A gate that cannot
read the state must not assume the state did not move.

⛔ **`volatile: true` says the step's input is not a file**: database rows, the
network, the clock. Such a step inherits no scope, is never remembered, and runs
every time. It is a separate word on purpose. A reconciliation step in a production
application asked twenty-four queries of a live schema while declaring a file scope
in the settings; the database can move without a single file moving, so the gate
skipped it as *"nothing it reads has changed"* and **counted it green while it was
red**. Declaring both a file scope and `volatile` is refused rather than resolved:
a reader could not tell which one the engine believed. So is declaring both
`state` and `volatile`: one says the input can be read, the other says it cannot.
Prefer `state` wherever the input can be printed — `volatile` is for what cannot,
such as a step that builds the database it measures.

⛔ **The cache says why it missed.** `X3_CACHE_WHY=1` makes a step that ran
again name the input that moved: `"architecture" ran again: dir ops/tmp/log
changed`. Without it, finding out why a step re-ran on a tree nobody touched
means guessing — and the answer is often that the run itself wrote into the tree
it measures. Measured in a production application: a step kept re-running
because the operator was redirecting the gate's own log into a directory that
step scans. That is not a cache fault, and no amount of reasoning about the
cache would have found it.

⛔ **A tree the walk does not enter can still be a dependency.** Fixture
directories (`testdata`) are skipped by every scan, but a test runs on top of
them: change a fixture and the answer changes. A step's memory therefore carries
the **content digest of every fixture tree** the run walked past, not just the
files it opened. Not entering and not depending on are different things, and
the first silently implied the second.

⛔ **An answer that came from a command, not a file, is an input too.** A step
that asks the version control system *"what changed?"* gets a list back, and when
that list comes back empty the step opens **no file at all** — its whole
observation is the settings it loaded. Nothing in the tree is then in the key, so
the step is remembered until a settings file moves, and in between a real change
goes unmeasured. Measured in a production application: a *documentation ↔ code*
step answered **green from the cache and red without it, on the same tree at the
same moment** — a finding the gate had lost. So the engine records the question
as well: every command it runs against the repository's history is stored as
*where it was asked, what was asked, and the digest of the answer*, and
validating a memory **asks it again**. The question is checked last, because a
file digest reads the disk while a question starts a process.

⛔ **A recorded question can only be a bare program name.** The memory is a text
file and the check runs what it finds in it; a name carrying a path separator is
neither recorded nor run.

⛔ **The file set is hashed once**, not per step, and only the files some step
declares. Measured in a production repository: 1 142 source files, 19.9 MB, hashed
in full in **206 ms** — which is also the whole of what a timestamp check could
save. A timestamp lies in both directions: a checkout refreshes it with the content
unchanged, and a copy carries an old one onto new content. The second is not a
slow gate, it is a green one that measured nothing.

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

<!-- x3-dist version=v0.245.0 capabilities=d9ca554260a11d0a48aa845570577bee9115e6128761ad5702b284c958dc88a3 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
