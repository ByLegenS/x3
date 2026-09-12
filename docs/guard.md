# The live world, before the command

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 guard`

**What it catches:** a long run started against the wrong live environment. The
guards run first, and the command after `--` starts only if they pass — one
process, one decision, no wrapper script.

```
x3 guard [-config <file>] [-report <file>] [-only <tags>] [-skip <tags>] [-stamp] -- <command> [args...]
```

`-report` is the only way a report is written: **stdout belongs to the launched
command**. The command is started with x3's **own environment and working
directory** — nothing added, removed or rewritten — and its exit code is
returned verbatim, so a launched test run behaves exactly as it would without
the guard in front of it.

### The decision rule

| Guards | Decision | What happens |
|---|---|---|
| all pass | `launch` | the command runs; x3 exits with its exit code |
| red, all of them `policy: warn` | `launch` | the command runs; each red is printed as `WARN` first |
| at least one red with `policy: block` | `blocked` | **the command is never started**; x3 exits `1` |

A guard that could not run at all — missing variable, unreachable host, unknown
driver — counts as red. **A live guard whose answer is unknown is not an
answer**, and the switch is fail-closed.

### Choosing which guards run

One file usually holds every guard a project has, but the gate that starts a
worker has no business waiting on a guard belonging to a different binary.
`tags` plus `-only` / `-skip` pick a subset **out of the same file**, so a
narrower run is still the file everybody reviews rather than a second copy that
drifts.

```json
{ "name": "database-reachable", "kind": "sql", "tags": ["db", "slow"],
  "dsnEnv": "APP_DSN", "query": "select 1", "equals": "1" }
```

| Written | What runs |
|---|---|
| neither flag | **every guard in the file** |
| `-only a,b` | guards carrying `a` or `b`, **plus every guard with no tags at all** |
| `-skip a` | everything except the guards carrying `a` |
| both | `-skip` wins on a guard that matches both |

**A guard with no tags always runs**: narrowing a set must not drop the check
nobody got round to classifying. Three selections are refused outright with exit
`2`, before any guard runs — a tag no guard carries (a misspelled `-skip` would
otherwise skip nothing and read as if it had), a selection that leaves no guard,
and an empty tag. Whatever a selection dropped is named in the report's
`skipped` list and in the stderr summary: **a check that did not run must never
look like a check that passed.**

### Exit codes

See **guard exit codes** in the [guard reference](guard-reference.md#guard-exit-codes).

`1` carries two meanings — "blocked" and "the command itself exited 1". The
report separates them: `decision` is `blocked` in the first case, and `launch`
with an `exit` field in the second.

## Live guards in `x3.json`

Guards are **declared, not coded**. There is no Go file per guard and no plugin:
the engine knows three general source kinds — `sql`, `http`, `exec` — plus
`steps`, and everything project-specific is data.

Validation inside `live` is **strict and up front**: an unknown key, a key
belonging to a different kind, a missing required key, a duplicate name, an
unknown policy or an empty guard list stops the run *before any guard executes*.

**Two surfaces, one law.** `//x3:live` in source code is a *marker* — this code
talks to a real provider. The `live` section is where runnable guards are
*defined*. Both are declared in a dictionary inside the engine, and in both an
entry the dictionary does not know turns the run red.

### Fields every guard has

See **guard fields every kind has** in the [guard reference](guard-reference.md#guard-fields-every-kind-has).

### `kind: "sql"`

See **guard fields for kind sql** in the [guard reference](guard-reference.md#guard-fields-for-kind-sql).

An expectation is mandatory here: a query with no expectation is answered by an
empty table. `sameRowsAs` is the third one, and the only one that reads a
**set**: [Two schemas, one question](guard-rows.md#two-schemas-one-question).

```json
{ "name": "schema-current", "kind": "sql", "policy": "block",
  "dsnEnv": "APP_DATABASE_URL",
  "query": "select max(version)::text from schema_migrations", "equals": "0117" }
```

### `kind: "http"`

See **guard fields for kind http** in the [guard reference](guard-reference.md#guard-fields-for-kind-http).

```json
{ "name": "provider-agent-enabled", "kind": "http", "policy": "warn",
  "url": "https://api.provider.example/v1/agents/self", "status": 200,
  "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
  "jsonPath": "/agent/permissions/0", "equals": "outbound" }
```

### `kind: "exec"`

See **guard fields for kind exec** in the [guard reference](guard-reference.md#guard-fields-for-kind-exec).

### `kind: "steps"` — a trial, not a reading

**What it catches:** a question that is not a fact you can read but an
experiment — *"does this module still compile once the application is removed
from the tree?"* Copy the tree, take the application out, build what is left,
put everything back. Without a multi-step kind the only way to write that is a
script inside the project, and a script is what x3 exists to remove: reviewed by
nobody, drifting when a path moves, never measured for whether it can still turn
red.

See **guard fields for kind steps** in the [guard reference](guard-reference.md#guard-fields-for-kind-steps).

A step takes `name`, `command`, `args`, `dir`, `env` (added to the inherited
environment for that step only), `output` (the same `must` / `mustNot` / `retry`
expectation the [work list](boxes.md#the-criteria) writes, read against both streams) and
`timeoutMs`. Without an `output`, a step's assertion is its **exit code**.

```json
{ "name": "the core still builds once the application is removed",
  "kind": "steps", "policy": "block",
  "workspace": { "copy": ["go.mod", "go.sum", "cmd", "internal", "core"],
                 "remove": ["internal/app"] },
  "steps": [
    { "name": "the core compiles with no application in the tree",
      "dir": "core", "command": "go", "args": ["build", "./..."],
      "env": { "GOWORK": "off" } },
    { "name": "the binary names a released core, not the working copy",
      "command": "go", "args": ["version", "-m", "out/service"],
      "output": { "must": ["example.com/core v0."], "mustNot": ["(devel)"] } } ] }
```

The copy is taken from the **working tree**, not from the last commit: if an
uncommitted change crossed the boundary, the trial should see it in the same
run. Three things are refused rather than run, each closing a way to a silent
green: a trial with **no steps** (it would pass every time), a `copy` path that
is **not on disk** (a build that fell because a source was missing is red for
the wrong reason), and a `remove` path that is **not there** (a trial measuring
an absence it never created is green by construction).

Absolute paths and `..` are refused everywhere in a trial — a `remove` that
climbed out of the copy would delete from the working tree, and no gate may
damage the thing it measures. **The working area is removed in every case**,
including when a step fails mid-way, and the paths inside it are stripped out of
the output that reaches the report: the person reading the red opens the file
**in the repository**, not a copy that no longer exists.

Removal on every exit path is not the whole of it: a **killed** run has no exit
path. Cancel the gate and the process tree closes with the copy still on disk
and nothing left to collect it. So sweeping the leftovers is the first act of
every trial — the same answer `testdb` reached for the same reason. The sweep
goes by **age**: a second x3 running right now has a working area under the same
name, and taking it would shoot a live trial in the foot. It reaches no other
name, and it cannot be switched off, because a sweep that can be switched off is
switched off the day it is inconvenient.

`write` is what makes the control experiment possible from the configuration
alone: the same trial with one file written into the copy has to go red, and a
trial whose red has never been seen is not a trial.

### Secrets never enter the report

Credentials are referenced **by environment variable name only**. Before
anything is written, those values are stripped out of the observed value **and
out of the error text** — driver errors routinely quote the DSN they failed on,
and that string is replaced with `[redacted]`. An empty variable is an error,
not an empty credential: the guard goes red instead of asking anonymously and
reporting a misleading `401`. `TestSecretNeverLeaves` makes a fake driver fail
with the DSN inside its own error message and asserts the password is nowhere in
the marshalled result.

## The guard report

```json
{ "version": 1, "config": "…/guard-block-red.json",
  "guards": [
    { "name": "impossible-platform", "kind": "exec", "policy": "block",
      "status": "fail", "expected": "equals \"there-is-no-such-platform\"",
      "observed": "windows", "detail": "observed value is not equal to the expected value" } ],
  "summary": { "pass": 1, "warned": 0, "blocked": 1 },
  "decision": "blocked", "command": ["x3", "scan", "internal"] }
```

See **the guard report fields** in the [guard reference](guard-reference.md#the-guard-report-fields).

**No timestamp unless you ask for one**: the same configuration and the same
answers must produce the same bytes.

## The command the run wraps

Guards run **before** the command and can say nothing about it. What is left is
the command's exit code — and that is half a criterion, because a runner that
finds nothing to run also exits zero. A gate step narrowed with a selector
(`go test -run Manifest`) keeps passing on the day its package empties out.

So the wrapped command's output is weighed too, against `live.command`:

```json
"live": { "guards": [ … ],
  "command": { "must": ["--- PASS"], "mustNot": ["no tests to run", "no test files"] } }
```

The engine does not know these sentences — **no runner is named in its source**;
the project writes what a real run of *its* command must say, in the words
work-list criteria already use (`must`, `mustNot`, `unmeasured`). The output
still streams to the terminal untouched; the copy kept for weighing is capped,
and a run that outgrows it is **not measured** rather than measured on half the
text. The command's own red passes through unchanged — an expectation renames
nothing — and `measured` carries the verdict: `pass`, `fail`, or `error`.

**Both directions are closed.** An expectation with no command to measure is a
dead expectation; a wrapped command with no expectation is a silent green. Both
stop the run. The only way past is the reason — `"unweighed": "<why it cannot be
weighed>"` written in place of `command` — and that reason is printed on every
run that wraps something, so an excused gate never becomes a quiet one. The
reason is outside the dead-expectation rule, because one configuration serves
runs that wrap a command and runs that only measure guards: an expectation a
command-less run can never satisfy is stale, while a reason it does not need is
simply not needed.

<!-- x3-dist version=v0.142.0 capabilities=26bb19f71c95ebb12c25cee2d5a14f374ec748b090b8afa420df48fd5ddb1db7 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
