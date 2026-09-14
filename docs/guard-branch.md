# A trial that asks and then chooses

[The pages](INDEX.md) - [what x3 is](../README.md)

## A trial that asks and then chooses

**What it catches:** the `if` that had to be written outside the gate, and the
variable that had to be written outside it. A trial runs its steps in order and
stops at the first one that does not hold, so *"is the schema already there, and
if it is not, put it there"* cannot be written at all: the reading that answers
the question is red exactly on the day something has to happen. The second half
is the same wall from the other side - what one step reads (a container id, a
token, a version) cannot reach the step after it. Both are why a project keeps a
shell script next to the gate, and a script is what x3 exists to remove.

### `when` - the answer a step waits for

A step may carry `when`: the name of an **earlier step of the same phase** and
the answer it waits for. The step runs only if the step it names ran and
answered that way.

Asking changes the step being asked about: **a measured red of a step that a
later `when` names does not end the trial.** It is an answer, and the trial goes
on down the other arm. There is deliberately no field that says *"swallow this
step's red"* - a red is turned into a branch by the reader that consumes it, so
a red nobody reads cannot be written.

Three fences, each closing a way to a silent green:

| | |
|---|---|
| a step that could not be **measured** at all | ends the trial anyway: a step that gave no answer cannot choose an arm, and an unmeasured arm counted as measured is this gate inverted |
| a step whose condition does not hold | **does not run**, and the report names it and says why; a skipped step is not a green one |
| a condition naming a step of the other phase | refused: the teardown runs whatever happened, and a condition reading across would ask about a step that may never have run |

```json
{ "name": "the schema is where the release expects it",
  "kind": "steps", "policy": "block",
  "steps": [
    { "name": "the migration is already applied", "command": "psql",
      "args": ["-tAc", "select count(*) from schema_migrations where version = '0118'"],
      "output": { "must": ["1"] } },
    { "name": "the migration is applied now", "when": { "step": "the migration is already applied", "holds": false },
      "command": "psql", "args": ["-v", "ON_ERROR_STOP=1", "-f", "migrations/0118.sql"] } ] }
```

```
step 1 of 2 "the migration is already applied" did not hold, and the trial went on:
1 step(s) did not run: step 2 of 2 "the migration is applied now":
its condition wanted "the migration is already applied" not to hold, and it held
```

### `capture` - the value the next step is handed

A step may carry `capture`: the **name** its trimmed output is handed on under.
Later steps read it as `${NAME}` in `args`, `env` values and `dir` - and so does
the **teardown**, because the row a trial wrote is taken away by the id the
measuring phase read. A condition reads its own phase only; a value crosses.

```json
{ "name": "the agent answers on a container of its own",
  "kind": "steps", "policy": "block",
  "steps": [
    { "name": "the container is up", "command": "docker",
      "args": ["run", "-d", "--rm", "example/agent:pinned"], "capture": "CONTAINER" },
    { "name": "it answers", "command": "docker",
      "args": ["exec", "${CONTAINER}", "/agent", "-ping"], "output": { "must": ["pong"] } } ],
  "after": [
    { "name": "the container is gone", "command": "docker", "args": ["kill", "${CONTAINER}"] } ] }
```

⛔ **The value never reaches the report.** A captured value can be a token or an
id, so it is removed from every sentence a run prints, exactly as a DSN is. Two
consequences are refused up front rather than discovered later: a guard cannot
weigh the output of a last step that hands it on (the expectation would be
compared against a removed text and could never hold), and a **capture nobody
reads** is refused - handing a value on puts it into no environment by itself,
so an unread one does nothing at all.

Four more configuration errors close the same door: a `when` that does not say
which answer it waits for, a `${NAME}` no earlier step captures, one name
captured twice, and an **empty** captured value - which is not a red but an
unmeasured run, because an argument silently emptied makes the step measure
something other than what is written.

<!-- x3-dist version=v0.220.0 capabilities=e4ce14129172cba712b0b7618de277b9b18687e55ac113928be89b82592f1fbc template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
