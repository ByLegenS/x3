# The gate itself, running in parallel

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 gate`

```
x3 gate [-config <file>] [-band fast|commit|full] [-only <names>] [-workers <n>] [-out <file>]
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

<!-- x3-dist version=v0.213.0 capabilities=f60b889e641897e52daf3f936af2ca5e03d88f986b31c320450e70c499a0c20a template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
