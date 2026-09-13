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
      "trials": [ { "say": "every file formatted", "run": ["gofmt", "-l", "."], "want": 0, "not": ".go" } ] },
    { "name": "language gate", "band": "fast",
      "trials": [
        { "say": "this repository", "run": ["{bin}", "lang", "."], "want": 0 },
        { "say": "a planted sample", "run": ["{bin}", "lang", "testdata/red"], "want": 1 }
      ] }
  ]
}
```

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


### The variables a step declares

`needs` says a step cannot run **without** a variable. `env` says what the step's
own environment **is**:

```json
{ "name": "schema drift", "band": "full",
  "env": { "APP_DEV_DSN": "${APP_DSN}" },
  "trials": [ { "run": ["{bin}", "guard", "-only", "schemadrift"], "want": 0 } ] }
```

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

<!-- x3-dist version=v0.193.0 capabilities=a59d3bbcaa3163d60c6fdd9b9e043fe2d17ace7a081b3baa2cd4e0c02241e7dc template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
