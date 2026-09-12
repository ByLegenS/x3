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
run that wraps something, so an excused gate never becomes a quiet one. It
excuses **weighing**, not the protection below; a runner announcing it had
nothing to run is still red under an `unweighed` configuration.

**One configuration, two kinds of run.** A project usually runs the same
configuration twice: wrapped around its build or test command, and on its own as
a plain health check. The second kind wraps nothing, so the dead-expectation
question — asked of a single run — would refuse it, and the project would end up
either not writing `command` at all (a silent green) or keeping a second
configuration file beside the first (and a configuration split in two is the one
where half of it goes stale). The way through is written down and justified:

```json
"live": { "guards": [ … ],
  "command": { "must": ["--- PASS"] },
  "unwrapped": "the same configuration also runs as a plain health check, with no command to wrap" }
```

`unwrapped` excuses **only** the dead-expectation question. The expectation stays
where it is: a run that *does* wrap a command still weighs its output, and a
command that breaks is still red with the expectation's own words. The reason is
printed on every command-less run, so an excused gate never becomes a quiet one.

**A runner that measured nothing does not make a green.** `go test -run <pattern>`
exits **0** when the pattern matches nothing, and so does a package with no test
file in it. That zero reads as "the work is done", and an expectation cannot be
relied on to catch it, because `unweighed` excuses writing one — measured on a
gate step whose packages had emptied out and burned green for months. So the
output is read for that announcement **whether or not one was written**:

```json
"live": { "blind": { "when": ["go test"], "says": ["no tests to run", "no test files"] } }
```

Both fields default to what is shown, so the block is normally left out. `when`
is matched against the command line **x3 was given**, not what a script inside it
goes on to run; only then are the sentences looked for. A command's own non-zero
exit passes through untouched — this looks only at greens, an expectation's own
red included, so "I measured and it failed" keeps its name — and the reading is a
**stream**, so a run that outgrows the copy kept for weighing is still watched.

**Every wrapper reads it, not just this one.** `x3 testdb run -- <command>` is a
gate step too, and nothing stands between its exit code and the gate, so the same
zero arrives there unweighed. It reads the same `live.blind` block from the same
file: two keys for one hole would be shaped in one place and forgotten in the
other. A `live` section is not required for it - the protection is a floor, not
something a project earns by writing guards - and `off` is printed there as well.

The default names a runner, which `must`/`mustNot` deliberately never do; the
difference is that it is a *default*. Another runner writes its own line
(`"when": ["pytest"], "says": ["no tests ran"]`), and an empty list in place of
either is refused — it cannot be told from an unwritten one, and would inherit
the default in silence. Removing the protection takes a reason
(`"blind": { "off": "…" }`), printed on every run that wraps a command, and `off`
cannot sit beside `when` or `says`: a protection is either shaped or removed.

**A dead excuse is red too.** `unwrapped` written with no `command` beside it
excuses nothing — a run that wraps no command was never in question — so the
configuration is refused: *"live.unwrapped is written but live.command is not"*.
That is what keeps the reason from outliving the expectation it was written for.
A blank reason is refused for the same reason a blank `unweighed` is.

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
